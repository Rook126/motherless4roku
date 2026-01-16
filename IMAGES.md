# Image Handling Guide

This document explains how images are managed, stored, and used in the Motherless4Roku application, as well as AI-powered image capabilities available through integrations.

## Image Assets Location

All image assets for the Roku channel are stored in the `Motherless4Roku/images/` directory. These images are bundled with the application package and referenced using the `pkg:/images/` URI scheme.

### Directory Structure

```
Motherless4Roku/images/
├── artwork/              # Content artwork and thumbnails
├── BG_dark_down.png      # Background image
├── MainMenu_Icon_*.png   # Main menu icons (HD/SD variants)
├── Overhang_*.png        # Overhang UI elements (logo, background)
├── VideoPlayer*.jpg      # Video player backgrounds
├── channel-poster_*.png  # Channel poster art (FHD/HD/SD)
└── splash-screen_*.jpg   # Splash screens (FHD/HD/SD)
```

## Image Requirements

### Supported Formats
- **PNG** - Recommended for UI elements, icons, and graphics requiring transparency
- **JPG/JPEG** - Recommended for photos, backgrounds, and content artwork

### Resolution Variants

The application supports multiple resolutions to accommodate different Roku device capabilities:

- **FHD (Full HD)**: 1920×1080 - For modern Roku devices
- **HD**: 1280×720 - For standard HD devices  
- **SD**: 720×480 - For older SD devices

### Required Channel Assets

The following images are required for channel certification and are referenced in `manifest`:

1. **Channel Posters** (for Roku home screen):
   - `channel-poster_fhd.png` (540×405 pixels)
   - `channel-poster_hd.png` (290×218 pixels)
   - `channel-poster_sd.png` (214×144 pixels)

2. **Splash Screens** (shown during channel launch):
   - `splash-screen_fhd.jpg` (1920×1080 pixels)
   - `splash-screen_hd.jpg` (1280×720 pixels)
   - `splash-screen_sd.jpg` (720×480 pixels)

3. **Overhang Elements** (top UI bar):
   - `Overhang_Logo_HD.png` (356×48 pixels recommended)
   - `Overhang_Background_HD.png` / `Overhang_Background_SD.png`

## Using Images in Components

Images are referenced in SceneGraph XML components using the `pkg:/` URI scheme:

```xml
<Overhang
  logoUri="pkg:/images/Overhang_Logo_HD.png"
  backgroundUri="pkg:/images/Overhang_Background_SD.png"/>
```

### In BrightScript Code

```brightscript
poster.uri = "pkg:/images/artwork/thumbnail.jpg"
```

## Adding or Modifying Images

### For Standard Assets

1. Place your image in the appropriate location under `Motherless4Roku/images/`
2. Follow the naming conventions (use resolution suffixes: `_fhd`, `_hd`, `_sd`)
3. Ensure proper file sizes and formats
4. Update references in XML components or BrightScript code as needed
5. Test on multiple device resolutions

### For Content Artwork

Content thumbnails and artwork should be placed in `Motherless4Roku/images/artwork/` and can be:
- Downloaded dynamically from content APIs
- Pre-bundled with the application
- Generated or processed through automation scripts

## AI-Powered Image Capabilities

This project integrates with the Hugging Face ecosystem for AI-powered image processing. See the [Hugging Face Integration Guide](docs/huggingface_integration.md) for details.

### Available Features

#### 1. Image Classification and Moderation

Automatically classify and moderate images before displaying them in the channel:

```typescript
import { HfInference } from "@huggingface/inference";

const inference = new HfInference(process.env.HF_TOKEN);

const classification = await inference.zeroShotImageClassification({
  model: "openai/clip-vit-large-patch14",
  inputs: {
    image: await fetch("https://example.com/image.jpg").then(res => res.blob()),
    candidate_labels: ["appropriate content", "inappropriate content"]
  }
});
```

#### 2. Image Generation

Generate custom artwork or promotional images:

```typescript
const image = await inference.textToImage({
  model: "black-forest-labs/FLUX.1-dev",
  provider: "replicate",
  inputs: "a promotional banner for a video channel"
});
```

#### 3. Automated Content Tagging

Use zero-shot classification to automatically tag content images for better organization and filtering within the Roku channel.

### Integration Workflow

1. **Pre-processing**: Use Node.js scripts to process images before bundling
2. **Moderation**: Filter inappropriate content using classification models
3. **Optimization**: Resize and compress images for optimal Roku performance
4. **Caching**: Store processed results to avoid repeated API calls

## Best Practices

1. **Optimize File Sizes**: Keep images as small as possible while maintaining quality
   - Use JPG for photos (quality 80-85%)
   - Use PNG only when transparency is needed
   - Compress all images before adding to the project

2. **Test Across Devices**: Always test image appearance on FHD, HD, and SD resolutions

3. **Use Consistent Naming**: Follow the project's naming conventions with resolution suffixes

4. **Version Control**: Commit images to the repository but consider using Git LFS for large assets

5. **Accessibility**: Ensure sufficient contrast for overlaid text and UI elements

6. **Content Rights**: Only use images you have rights to distribute

## Troubleshooting

### Images Not Displaying
- Verify the file path uses the correct `pkg:/images/` prefix
- Check that the image file exists in the correct location
- Ensure the image format is supported (PNG or JPG)
- Verify file permissions allow reading

### Poor Image Quality
- Use resolution-appropriate variants (FHD/HD/SD)
- Check compression settings (avoid over-compression)
- Ensure source images are high enough resolution

### Performance Issues
- Reduce image file sizes through compression
- Use appropriate formats (JPG for photos, PNG for graphics)
- Limit the number of images loaded simultaneously

## Additional Resources

- [Roku Development Guidelines](https://developer.roku.com/docs/developer-program/getting-started/roku-dev-prog.md)
- [Hugging Face Integration Guide](docs/huggingface_integration.md) - AI-powered image processing
- [SceneGraph Documentation](https://developer.roku.com/docs/developer-program/core-concepts/scenegraph.md)
