# Generative Pretraining for Deep Neural Networks on MNIST

This project studies the use of **Restricted Boltzmann Machines (RBMs)** and **Deep Belief Networks (DBNs)** for unsupervised pretraining before supervised fine-tuning of a **Deep Neural Network (DNN)** on the **MNIST** dataset.

The work is organized in three main parts:
1. validation of RBM and DBN implementations on the **Binary AlphaDigits** dataset,
2. application of **DBN pretraining** to initialize DNNs for **MNIST classification**,
3. bonus comparison of generative models on MNIST, including **VAE**, **GAN**, and **DDPM**.

## Main objectives

The project aims to:
- implement RBMs, DBNs, and DNNs from scratch,
- evaluate whether unsupervised pretraining improves optimization and classification performance,
- study the effect of:
  - network depth,
  - hidden-layer width,
  - training-set size,
- compare several generative models qualitatively on MNIST.

## Repository structure

```text
generative-pretraining-mnist/
├── README.md
├── requirements.txt
├── .gitignore
├── notebooks/
│   └── final_notebook.ipynb
├── results/
├── report/
│   └── final_report.pdf
└── data/
