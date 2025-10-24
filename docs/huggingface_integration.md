# Hugging Face Hub Integration Guide

This project can be extended with content that is generated or moderated by machine learning models hosted on the [Hugging Face Hub](https://huggingface.co/docs). The examples below demonstrate how to manage models programmatically and how to invoke different inference providers using the [Hugging Face Inference Endpoints JavaScript SDK](https://huggingface.co/docs/huggingface.js/inference/overview).

## Prerequisites

- Node.js 18 or newer.
- An access token with the appropriate permissions stored in the `HF_TOKEN` environment variable.
- The `@huggingface/inference` and `@huggingface/hub` packages installed in your project:

```bash
npm install @huggingface/hub @huggingface/inference
```

- (Optional) A `.env` file loaded by your build tooling so that Roku-side ingestion scripts can reference `HF_TOKEN` without hard-coding credentials.

> **Tip:** When shipping Node utilities alongside a Roku channel, keep them in a sibling folder (for example, `scripts/automation`). That avoids bundling the dependencies into the BrightScript application package.

## Configure the Environment

1. Create an access token from the Hugging Face profile page with the `Read` and `Write` scopes that match your needs.
2. Store the token locally, for example with `echo "HF_TOKEN=hf_xxx" >> .env` and load it via [`dotenv`](https://www.npmjs.com/package/dotenv) in any helper scripts.
3. For CI/CD, inject the same token as a secret so automated sync jobs can push model changes or pre-generate assets.

## Create or Update a Model Repository

Use the Hub client to create a repository or upload assets such as weights and configuration files. The snippet below assumes that `HF_TOKEN` is set in your environment and that the caller has access to the `my-user` namespace.

```ts
import { createRepo, uploadFile } from "@huggingface/hub";

await createRepo({
  repo: { type: "model", name: "my-user/nlp-model" },
  accessToken: process.env.HF_TOKEN!,
});

await uploadFile({
  repo: "my-user/nlp-model",
  accessToken: process.env.HF_TOKEN!,
  file: {
    path: "pytorch_model.bin",
    // Replace with a reference to a real File, Blob, or Buffer instance.
    content: new Blob([/* model binary data */]),
  },
});
```

## Run Inference Across Multiple Providers

The Inference SDK can route requests to a range of partner providers without changing your application logic. Specify the `provider` field to select the infrastructure that should serve the request.

```ts
import { HfInference } from "@huggingface/inference";

const inference = new HfInference(process.env.HF_TOKEN);

const chatResponse = await inference.chatCompletion({
  model: "meta-llama/Llama-3.1-8B-Instruct",
  provider: "sambanova", // also supports together, fal-ai, replicate, cohere, etc.
  messages: [
    { role: "user", content: "Hello, nice to meet you!" },
  ],
  max_tokens: 512,
  temperature: 0.5,
});

const image = await inference.textToImage({
  model: "black-forest-labs/FLUX.1-dev",
  provider: "replicate",
  inputs: "a picture of a green bird",
});
```

### Handling Responses

- `chatResponse` contains the generated messages plus usage metadata. Persist or display the assistant response by reading `chatResponse.choices[0].message.content`.
- `image` resolves to binary image data (for example, a `Blob`). To use it in a Roku application, write the blob to disk and load it as a texture or upload it to a CDN.

## Moderate Wardrobe Galleries (e.g., Police Outfits)

Uniform-focused galleries often need an automated review pass before they ship to production. The zero-shot image classification endpoint can tag incoming assets without building a custom model.

```ts
import { HfInference } from "@huggingface/inference";

const inference = new HfInference(process.env.HF_TOKEN);

const classification = await inference.zeroShotImageClassification({
  model: "openai/clip-vit-large-patch14",
  inputs: {
    image: await fetch("https://example.com/uploads/police-outfit.jpg").then((res) => res.blob()),
    candidate_labels: [
      "police uniform",
      "casual wear",
      "swimwear",
      "non-uniform costume",
    ],
  },
});

const topLabel = classification.sort((a, b) => b.score - a.score)[0];
if (topLabel.label !== "police uniform") {
  // Flag the asset for manual review or hide it from the Roku feed.
}
```

Because the call returns scores per label, you can decide whether to filter, re-route, or queue the asset based on its likelihood of matching the expected wardrobe style.

## Operational Tips

- Rotate your access tokens regularly and prefer project-scoped tokens when possible.
- When uploading large model files, enable resumable uploads via the optional `onProgress` callback exposed by `uploadFile`.
- Monitor provider-specific quotas and rate limits when mixing hosted inference back ends.
- Cache inference results where possible to reduce latency and cost for repeated requests.
- Persist moderation decisions (for example, store the `topLabel` example above) so the Roku channel can filter feeds without calling the API repeatedly.

These patterns allow you to weave Hugging Face–powered workflows into the content pipelines that feed the Roku channel.
