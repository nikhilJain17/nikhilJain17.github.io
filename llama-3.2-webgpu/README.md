---
title: Llama 3.2 WebGPU
emoji: 🦙
colorFrom: green
colorTo: pink
sdk: static
pinned: false
license: apache-2.0
models:
  - onnx-community/Llama-3.2-1B-Instruct-q4f16
short_description: A powerful AI chatbot that runs locally in your browser
thumbnail: https://huggingface.co/spaces/webml-community/llama-3.2-webgpu/resolve/main/banner.png
---

# Llama-3.2 WebGPU

A simple React + Vite application for running [Llama-3.2-1B-Instruct](https://huggingface.co/onnx-community/Llama-3.2-1B-Instruct-q4f16), a powerful small language model, locally in the browser using Transformers.js and WebGPU-acceleration.

## Getting Started

Follow the steps below to set up and run the application.

### 1. Navigate to the Project Directory

From this blog repository's root:

```sh
cd llama-3.2-webgpu
```

### 2. Install Dependencies

Install the necessary dependencies using npm:

```sh
npm ci
```

### 3. Run the Development Server

Start the development server:

```sh
npm run dev
```

Open `http://localhost:5173/transformers/` (or the URL printed by Vite if
that port is occupied) in a browser with WebGPU support. Select `q4f16` or
`fp16`, then load the model. The first load downloads weights from
`onnx-community/Llama-3.2-1B-Instruct-ONNX` on Hugging Face; inference runs in
your browser. Keep the terminal running and press `Ctrl+C` to stop.

This demo runs separately from the Hugo blog server. It was restored from
commit `d941989`, including the precision selector and prefill/decode TPS
measurements. The blog deployment workflow does not build or publish it.

To check the production build locally, run `npm run build`.
