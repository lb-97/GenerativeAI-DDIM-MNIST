# GenerativeAI-DDIM-MNIST

The first breakthrough paper on Diffusion models is by DDPM - Denoising Diffusion Probabilistic models. Inspired by previous work on nonequilibrium thermodynamics, they show that training of diffusion models while maximizing the posterior likelihood in an image generation task is mathematically equivalent to denoising score matching. In simple terms, there are two processes in diffusion modelling - forward & reverse. Forward process iteratively produces noisy images using noise schedulers. This can be reduced to one step noisy image through reparametrization technique. During training in the reverse process a U-Net is trained to estimate the noise in the final noisy image. During inference/sampling, noise is iteratively estimated and removed from a random noisy image to generate a new unseen image. The L2 loss used to estimate the noise during training is mathematically equivalent to maximizing the posterior likelihood i.e., maximizing the distribution of final denoised image. You can find more details in this paper. Stable Diffusion paper moves the needle by making diffusion model more accessible, scalable, trainable using a single Nvidia A100 GPU. Earlier diffusion models were difficult to train requiring 100s of training days, instability issues and restricted to image modality. Stable Diffusion achieved training stability with conditioning on multimodal data by working in latent space. A pre-trained image encoder such as that of VQ-VAE is used to downsample and extract imperceptible details of an input image. These latents are used to trained Diffusion model discussed above. Doing so separates the notion of perceptual compression and generative nature of the whole network. Later the denoised latents can be passed through a VQ-VAE trained decoder to reconstruct images in pixel space. This results in lesser complex model, faster training and high quality generative samples.

Currently I'm looking for unconditional image reconstruction/denoising/generation using SD. **I completed putting together keras implementation of unconditional SD. Since I couldn't find official implementation of unconditional SD code, I collated DDPM diffusion model codebase, VQ-VAE codebase separately.** DDPM code uses Attention based U-Net for noise prediction. The basic code blocks of the U-Net are ResidualBlock & AttentionBlock. ResidualBlock is additionally conditioned on the diffusion timestep, DDPM implements this conditioning by adding diffusion timestep to the input image, whereas DDIM performs a concatenation. Downsampling & Upsampling in the U-Net are performed 4 times with decreasing & increasing widths respectively. Each downsampling layer consists of two ResidualBlocks, an optional AttentionBlock and a convolutional downsampling(stride=2) layer. At each upsampling layer, there's a concatenation from respective downsampling layer, three ResidualBlocks, an optional AttentionBlock, keras.layers.Upsampling2D and a Conv2D layers. The Middle layer consists of two ResidualBlocks with an AttentionBlock in between resulting in no change in the output size. The final output of Upsampling layer is followed by a GroupNormalization layer, Swish Activation layer and Conv2D layer to provide an output with desired dimensions.

Continuing on my MNIST experiements, I ran into Multi Distribution issues while training unconditional Diffusion Model(DM). Without getting into too many details I can summarize that having a custom train_step function in tensorflow, without any default loss reduction such as tf.reduce_mean or tf.keras.losses.Reduction.SUM, requires more work than model.fit(). So, my current loss function used for training DM is reduced on the last channel while the rest of the shape of each batch is kept intact. When using distributed training, tensorflow requires the user to take care of gradient accumulation if it's an unreduced loss. So, I tried to learn from Tensorflow tutorials. Alas, all their multidistributed strategy examples were based on functional API models whereas my approach is based on object oriented implementation. This led to design issues. For the sake of time management, I did a little bit of tweaking. While compiling the model under tf.distribute.MirroredStrategy, I passed tf.keras.losses.Reduction.SUM parameter to the loss function and divided the loss by a pre-decided factor which is np.prod(out.shape[:-1]) i.e., number of elements in the output shape excluding the last channel which is reduced in the loss function. This tweak worked and also does not have any unexpected impacts on the architectue as well as the training paradigm.

I followed the architecture described above for the DM model. I trained this on VQ-VAE latents of MNIST dataset for 200 diffusion steps, 2 Nvidia V100 GPUs, Adam Optimizer with 2e-4 learning rate, 200 batch size per GPU for 100+ epochs. For the generative process, I denoised random samples for 50, 100 and 200 steps on the best performing model(112 epochs). Here are the results I achieved -

![200 Diffusion Steps on MNIST](https://github.com/lb-97/dipy/blob/blog_branch_week5/doc/posts/2023/assets/DM-MNIST-112epoch.png)

We see some resembalnce of digit shapes in the generated outputs. On further training for 300 diffusion timesteps for the best performing model( 108 epochs) with least training loss, the visuals have improved drastically -

![300 Diffusion Steps on MNIST](https://github.com/lb-97/dipy/blob/blog_branch_week5/doc/posts/2023/assets/DM-MNIST-DDIM300-108epoch.png)

These outputs show the effectiveness of the model architecture, training parameters and the codebase.

