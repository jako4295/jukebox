**Status:** Archive (code is provided as-is, no updates expected)

# Jukebox
Code for "Jukebox: A Generative Model for Music"

[Paper](https://cdn.openai.com/jukebox.pdf) 
[Blog](https://openai.com/blog/jukebox) 
[Explorer](http://jukebox.openai.com/) 
[Colab](https://colab.research.google.com/drive/1IQcUNjxLs79YVKLF-A8ghFLCaVYHjc4v)

# Install
``` 
# Required: Sampling
conda create --name jukebox python=3.7.5
conda activate jukebox
conda install mpi4py=3.0.3
conda install pytorch=1.4 torchvision=0.5 cudatoolkit=10.0 -c pytorch
cd jukebox
pip install -r requirements.txt
pip install -e .

# Required: Training
conda install av=7.0.01 -c conda-forge 
cd tensorboardX
python setup.py install
 
# Optional: Apex for faster training with fused_adam
conda install pytorch=1.1 torchvision=0.3 cudatoolkit=10.0 -c pytorch
cd apex
pip install -v --no-cache-dir --global-option="--cpp_ext" --global-option="--cuda_ext" ./
```

# Sampling
To sample, run the following command. Model can be `5b`, `5b_lyrics`, `1b_lyrics`
``` 
python jukebox/sample.py --model=5b_lyrics --name=sample_5b --levels 3 --sample_length_in_seconds 20 --total_sample_length_in_seconds 180 --sr 44100 --n_samples 6 --hop_fraction 0.5,0.5,0.125
```
``` 
python jukebox/sample.py --model=1b_lyrics --name=sample_1b --levels 3 --sample_length_in_seconds 20 --total_sample_length_in_seconds 180 --sr 44100 --n_samples 16 --hop_fraction 0.5,0.5,0.125
```
The above generates the first `sample_length_in_seconds` seconds of audio from a song of total length `total_sample_length_in_seconds`.

On a V100, it takes about 3 hrs for 5b model to fully sample the above 20 seconds of music. Since this could be a long time, its recommended to use n_samples > 1 so you can generate multiple samples in parallel.   

The samples decoded from each level are stored in `{name}/level_{level}`. You can also view the samples as an html with the aligned lyrics under `{name}/level_{level}/index.html`.

# Training
## VQVAE
To train a small vqvae, run
```
mpiexec -n {ngpus} python jukebox/train.py --hps=small_vqvae --name=small_vqvae --sample_length 262144 --bs 4 --nworkers 4 --audio_files_dir {audio_files_dir} --labels False --train --aug_shift --aug_blend
```
Here, `{audio_files_dir}` is the directory in which you can put the audio files for your dataset, and `{ngpus}` is number of GPU's you want to use to train. 
The above trains a two-level VQ-VAE with `downs_t = (5,3)`, and `strides_t = (2, 2)` meaning we downsample the audio by `2**5 = 32` to get the first level of codes, and `2**8 = 256` to get the second level codes.  
Checkpoints are stored in the `logs` folder. You can monitor the training by running Tensorboard
```
tensorboard --logdir logs
```
    
## Prior
### Train prior or upsamplers
Once the VQ-VAE is trained, we can restore it from its saved checkpoint and train priors on the learnt codes. 
To train the top-level prior, we can run

```
mpiexec -n {ngpus} python jukebox/train.py --hps=small_vqvae,small_prior,all_fp16,cpu_ema --name=small_prior --sample_length 2097152 --bs 4 --nworkers 4 --audio_files_dir {audio_files_dir} --labels False --train --test --aug_shift --aug_blend --restore_vqvae logs/small_vqvae/checkpoint_latest.pth.tar --prior --levels 2 --level 1 --weight_decay 0.01 --save_iters 1000
```

To train the upsampler, we can run
```
mpiexec -n {ngpus} python jukebox/train.py --hps=small_vqvae,small_upsampler,all_fp16,cpu_ema --name=small_upsampler --sample_length 262144 --bs 4 --nworkers 4 --audio_files_dir {audio_files_dir} --labels False --train --test --aug_shift --aug_blend --restore_vqvae logs/small_vqvae/checkpoint_latest.pth.tar --prior --levels 2 --level 0 --weight_decay 0.01 --save_iters 1000
```
We pass `sample_length = n_ctx * downsample_of_level` so that after downsampling the tokens match the n_ctx of the prior hps. 
Here, `n_ctx = 8192` and `downsamples = (32, 256)`, giving `sample_lengths = (8192 * 32, 8192 * 256) = (65536, 2097152)` respectively for the bottom and top level. 

### Reuse pre-trained VQ-VAE and retrain top level prior on new dataset.
Our pre-trained VQ-VAE can produce compressed codes for a wide variety of genres of music, and the pre-trained upsamplers can upsample them back to audio that sound very similar to the original audio.
To re-use these for a new dataset of your choice, you can retrain just the top-level  

To retrain top-level on a new dataset, run
```
mpiexec -n {ngpus} python jukebox/train.py --hps=vqvae,small_prior,all_fp16,cpu_ema --name pretrained_vqvae_small_prior --sample_length=1048576 --bs 4 --nworkers 4 --bs_sample 4 --aug_shift --aug_blend --audio_files_dir {audio_files_dir} --labels False --train --test --prior --levels 3 --level 2 --weight_decay 0.01 --save_iters 1000
```

You can then run sample.py with the top-level of our models replaced by your new model. To do so, add an entry 'my_model' in MODELs (in make_models.py) with the (vqvae hps, upsampler hps, top-level prior hps) of your new model, and run sample.py with `--model=my_model` . 
	

# Citation

Please cite using the following bibtex entry:

```
@article{dhariwal2020jukebox,
  title={Jukebox: A Generative Model for Music},
  author={Dhariwal, Prafulla and Jun, Heewoo and Payne, Christine and Kim, Jong Wook and Radford, Alec and Sutskever, Ilya},
  journal={arXiv preprint arXiv:[TODO]},
  year={2020}
}
```