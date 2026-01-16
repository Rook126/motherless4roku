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

## Image Editing Workflows

When working with image editing requests, especially those that aim to preserve character identity while modifying other aspects (e.g., changing wardrobe, background, or pose), it's essential to have proper inputs and clear requirements before proceeding.

### Requirements for Image Editing Tasks

To successfully edit an image while preserving character features, you need:

1. **Source Image**: The original image file containing the character or subject to be edited. This should be provided as:
   - A URL pointing to an accessible image resource
   - A local file path or File/Blob object
   - Base64-encoded image data

2. **Edit Instructions**: Clear description of what should be changed, such as:
   - "Change the outfit to a police uniform"
   - "Replace the background with a beach scene"
   - "Modify the pose to standing with arms crossed"

3. **Preservation Constraints**: Specification of what should remain unchanged:
   - Face and facial features
   - Body proportions
   - Skin tone and hair color
   - Overall character identity

### Handling Missing Image Inputs

If a user requests image editing without providing the source image, you should:

1. **Prompt for the Image**: Request that the user provides the image through one of the supported methods. Example responses:
   ```
   "I don't see any images provided. To perform image editing, please share the actual 
   image file you'd like to edit (as a URL, file upload, or blob) along with details 
   about what changes you'd like to make."
   ```
   
   Or for a more specific context:
   ```
   "I can't perform image editing without a source image. Could you share the image 
   you'd like to modify? Please also let me know what specific edits you need 
   (e.g., wardrobe change, background replacement, style adjustment)."
   ```

2. **Clarify the Task**: Ask specific questions about:
   - What type of editing is needed (wardrobe change, background replacement, style transfer, etc.)
   - Which elements should be preserved (character identity, pose, lighting, etc.)
   - What the desired outcome looks like

3. **Provide Alternative Solutions**: If the image cannot be provided, consider:
   - Text-to-image generation with detailed character descriptions
   - Reference image search to find similar content
   - Describing the workflow so the user can implement it when they have the image

### Example: Image Editing with Character Preservation

For models that support image-to-image editing with instruction following (such as InstructPix2Pix-style models), you can structure requests like this:

```ts
import { HfInference } from "@huggingface/inference";

const inference = new HfInference(process.env.HF_TOKEN);

// Ensure the image is provided before proceeding
if (!sourceImageBlob) {
  throw new Error("Source image is required for editing. Please provide the image to edit.");
}

const editedImage = await inference.imageToImage({
  model: "timbrooks/instruct-pix2pix",
  inputs: sourceImageBlob,
  parameters: {
    prompt: "Change the outfit to a police uniform while keeping the same person and pose",
    negative_prompt: "different person, different face, distorted features",
    num_inference_steps: 50,
    image_guidance_scale: 1.5,
    guidance_scale: 7.5,
  },
});
```

### Best Practices

- **Validate Inputs Early**: Check that all required images and parameters are provided before making API calls
- **Set Clear Expectations**: Communicate what level of character preservation is achievable with the chosen model
- **Handle Errors Gracefully**: Provide helpful error messages when inputs are missing or invalid
- **Test with Sample Images**: Verify that your editing pipeline works with representative test cases before deploying

## Operational Tips

- Rotate your access tokens regularly and prefer project-scoped tokens when possible.
- When uploading large model files, enable resumable uploads via the optional `onProgress` callback exposed by `uploadFile`.
- Monitor provider-specific quotas and rate limits when mixing hosted inference back ends.
- Cache inference results where possible to reduce latency and cost for repeated requests.
- Persist moderation decisions (for example, store the `topLabel` example above) so the Roku channel can filter feeds without calling the API repeatedly.

These patterns allow you to weave Hugging Face–powered workflows into the content pipelines that feed the Roku channel.
