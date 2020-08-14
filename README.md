# high-fidelity-generative-compression
Pytorch implementation of the paper ["High-Fidelity Generative Image Compression" by Mentzer et. al.](https://hific.github.io/)

## Warning
This is a preliminary version. There may be sharp edges.

## Details
This repository defines a model for learnable image compression. There are three main components to this model, as described in the original paper:

1. An autoencoding architecture defining a nonlinear transform to latent space. This is used in place of the linear transforms used by traditional image codecs.
2. A hierarchical (two-level in this case) entropy model over the quantized latent representation enabling lossless compression through standard entropy coding.
3. A generator-discriminator component that encourages the decoder/generator component to yield realistic reconstructions.

The model is then trained end-to-end by optimization of a modified rate-distortion Lagrangian. 

## Note
The generator is trained to achieve realistic and not exact reconstruction. Therefore, in theory **images which are compressed and decoded may be arbitrarily different from the input**. This precludes usage for sensitive applications. An important caveat from the authors is reproduced here: 

> "_Therefore, we emphasize that our method is not suitable for sensitive image contents, such as, e.g., storing medical images, or important documents._" 

![guess](assets/recon.jpg)

<details>
  <summary>Guess which half is the reconstruction? (Click side arrow to reveal) </summary>

> Bottom row, (average bpp 0.097) v. the top row JPG originals (average bpp 2.981).

</details>

## Usage
* Install Pytorch nightly and dependencies from [https://pytorch.org/](https://pytorch.org/). Then install other requirements.
```
pip install -r requirements.txt
```
* Download a large (> 100,000) dataset of reasonably diverse color images. We found that using 1-2 training divisions of the [OpenImages](https://storage.googleapis.com/openimages/web/index.html) dataset was able to produce satisfactory results. Add the dataset path to the `DatasetPaths` class in `default_config.py`.
* Clone this repository, `cd` in and view command line options/default arguments.
```
git clone https://github.com/Justin-Tan/high-fidelity-generative-compression.git
cd high-fidelity-generative-compression

vim default_config.py
python3 train.py -h
```

### Training
* For best results, as described in the paper, train an initial base model using the rate-distortion loss only, together with the hyperprior model, e.g. to target low bitrates:
```
python3 train.py --model_type compression --regime low --n_steps 1e6
```

* Then use the checkpoint of the trained base model to 'warmstart' the GAN architecture. Training the generator and discriminator from scratch was found to result in unstable training, but YMMV.
```
python3 train.py --model_type compression_gan --regime low --n_steps 1e6 --warmstart --ckpt path/to/base/checkpoint
```
* Training for 1e5 steps using a batch size of 16 was sufficient to get reasonable results at sub-0.01 `bpp` on average. If you get out-of-memory errors try reducing the number of residual blocks in the generator (default 7, the original paper used 9), decreasing the batch size, or training on smaller crops (default `256 x 256`).

### Compression
* To obtain a _theoretical_ measure of the bitrate under some trained model, run `compress.py`. This will report the bits-per-pixel attainable by the compressed representation (`bpp`) and other fun metrics and perform a forward pass through the model to obtain the reconstructed image.
```
python3 compress.py --img path/to/your/image --ckpt path/to/trained/model
```
* The reported `bpp` is the theoretical bitrate required to losslessly store the quantized latent representation of an image as determined by the learned probability model provided by the hyperprior using some entropy coding algorithm. Comparing this (not the size of the reconstruction) against the original size of the image will give you an idea of the reduction in memory footprint. This repository does not currently support actual compression to a bitstring ([TensorFlow Compression](https://github.com/tensorflow/compression) does this well). We're working on an ANS entropy coder to support this in the future.

### Notes
* The "size" of the compressed image as reported in `bpp` does not account for the size of the model required to decode the compressed format.
* The total size of the model is around 737 MB. Forward pass time should scale sublinearly provided everything fits in memory.

### Contributing
All content in this repository is licensed under the Apache-2.0 license. Feel free to submit any corrections or suggestions as issues.

<!-- ### Known Issues / Todo
* Training is unstable for high bitrate models (passing the `--regime high` flag in `train.py`). Currently unsure whether this is due to the dataset, or a flaw in the model. -->

### Acknowledgements
* The code under `hific/perceptual_similarity/` implementing the perceptual distortion loss is modified from the [Perceptual Similarity repository](https://github.com/richzhang/PerceptualSimilarity).
<!-- * Kookaburra image (`data/kookaburra.jpg`) by [u/Crispy_Chooken](https://old.reddit.com/r/australia/comments/i3ffpk/best_photo_of_a_kookaburra_ive_taken_yet/).
* The cat in the main image is my neighbour's. -->

### Authors
* Grace Han
* Justin Tan

### References
The following additional papers were useful to understand implementation details.
1. Johannes Ballé, David Minnen, Saurabh Singh, Sung Jin Hwang, Nick Johnston. Variational image compression with a scale hyperprior. [arXiv:1802.01436 (2018)](https://arxiv.org/abs/1802.01436)
2. David Minnen, Johannes Ballé, George Toderici. Joint Autoregressive and Hierarchical Priors for Learned Image Compression. [arXiv 1809.02736 (2018)](https://arxiv.org/abs/1809.02736)
3. Johannes Ballé, Valero Laparra, Eero P. Simoncelli. End-to-end optimization of nonlinear transform codes for perceptual quality. [arXiv 1607.05006 (2016)](https://arxiv.org/abs/1607.05006)

## Citation
This is not the official implementation. Please cite the [original paper](https://arxiv.org/abs/2006.09965) if you use their work.
```
@article{mentzer2020high,
  title={High-Fidelity Generative Image Compression},
  author={Mentzer, Fabian and Toderici, George and Tschannen, Michael and Agustsson, Eirikur},
  journal={arXiv preprint arXiv:2006.09965},
  year={2020}
}
```
