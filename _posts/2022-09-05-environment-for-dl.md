---
title: 'Set up environment for deep learning on Mac M1'
date: 2022-09-05
permalink: /posts/2022/09/Set-up-environment-for-deep-learning-on-Mac-M1/
tags:
  - machine learning
---

This article introduce steps to set up the Python environment for deep learning on Mac M1. We will cover
- Install Miniforge 3
- Install Tensorflow
- Install Transformer

Before we start, if you have already installed Anaconda or other conda environment, you may want to uninstall them first using `anaconda-clean`.

    conda install anaconda-clean
    anaconda-clean --yes

To remove the entire directory, try following.

    rm -rf ~/anaconda3
    rm -rf ~/.anaconda_backup 
    rm -rf ~/opt/anaconda3
    rm -rf ~/miniforge3

Finally, we need to remove anaconda path in .bash_profile. `open .bash_profile` from terminal, delete the line `export PATH="/Users/your_username/anaconda3/bin:$PATH"`. If you don't have permission to write to it, use `sudo chown your_username ~/.bash_profile` to change the owner of the file, and try again.

Now we could start the main course.

### 1. Install Miniforge 3
Download miniforge3 from [here](https://github.com/conda-forge/miniforge/#download). Choose Miniforge3-MacOSX-arm64，and save to Downloads. 

![miniforge3](/images/blog/2022-09-05-environment-for-dl/miniforge3.png)

Run the command below to install. Follow the default option during installation. 

    chmod +x ~/Downloads/Miniforge3-MacOSX-arm64.sh
    sh ~/Downloads/Miniforge3-MacOSX-arm64.sh
    source ~/miniforge3/bin/activate

You could also use homebrew `brew install miniforge`.

After you have successfully install Miniforge 3, restart the terminal and move to the next step.

### 2. Install Tensorflow
Create a conda environment for Tensorflow.

    conda create -n tensorflow python==3.9
    conda activate tensorflow

Note that `tensorflow` after `conda create -n` is the name for this new environment, feel free to use any name you like. I realized at the end that I put Tensorflow and Transformers under one environment and named it "tensorflow" :sweat_smile:. You could always rename it using `conda rename -n old_name -d new_name`.

Install tensorflow。

    conda install -c apple tensorflow-deps  
    python -m pip install tensorflow-macos
    python -m pip install tensorflow-metal 

To check if the installation is successful, run `conda env list`, you will find two environments, base and tensorflow. You could also check available kernel by `jupyter kernelspec list`.

![conda_env](/images/blog/2022-09-05-environment-for-dl/conda_env.png)

Next, let's install some commonly-used packages. Note that the first two lines set `conda-forge` as the highest priority channel.

    conda config --add channels conda-forge
    conda config --set channel_priority strict
    conda install jupyter jupyterlab pandas matplotlib seaborn nltk spacy beautifulsoup4 scikit-learn

Finally, open the jupyter lab to check your Tensorflow version and device. You can now try Tensorflow on your mac! 

    source ~/miniforge3/bin/activate
    conda activate tensorflow
    jupyter lab

![jupyter_notebook](/images/blog/2022-09-05-environment-for-dl/jupyter_notebook.png)

### 3. Install Transformers
Before installing Transformers, we need PyTorch and Rust.

    pip install -U --pre torch torchvision torchaudio --extra-index-url https://download.pytorch.org/whl/nightly/cpu
    curl — proto ‘=https’ — tlsv1.2 -sSf https://sh.rustup.rs | sh

Now you can run `pip install transformers` to get the job done. Enjoy your journey!


### Reference
- [Anaconda](https://docs.anaconda.com/anaconda/install/uninstall/), Uninstalling Anaconda Distribution
- [Apple developer](https://developer.apple.com/metal/tensorflow-plugin/)
- [better data science](https://betterdatascience.com/install-tensorflow-2-7-on-macbook-pro-m1-pro/), How To Install TensorFlow 2.7 on MacBook Pro M1 Pro With Ease
- [James Briggs](https://jamescalam.medium.com/hugging-face-and-sentence-transformers-on-m1-macs-4b12e40c21ce), Hugging Face and Sentence Transformers on M1 Macs
- [Jeff Heaton @ Youtube](https://www.youtube.com/watch?v=w2qlou7n7MA), Mac M1 Monterey Installing Miniforge and Anaconda/Miniconda Side-by-Side 