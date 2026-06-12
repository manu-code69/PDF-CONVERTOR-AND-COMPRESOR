## 🔒 Data Security & Privacy

Your files remain completely under your control throughout the conversion process.

### Privacy First

* Files are processed locally on the server running this application.
* No files are sent to third-party services.
* No cloud storage is used.
* Internet access is not required for conversion or compression.
* Uploaded files are automatically deleted after processing.
* Only the generated output PDF is temporarily stored for download.

### Security Workflow

```text
Your File
    ↓
Local Processing
    ↓
PDF Conversion / Compression
    ↓
Output Generated
    ↓
Upload Removed
```

This ensures that sensitive documents such as invoices, contracts, reports, certificates, and personal records remain private during processing.

---

# PDF Compression Logic

The application compresses PDFs by converting each page into an optimized image and rebuilding the PDF using progressively lower resolutions until the desired file size is reached.

Unlike traditional PDF optimization tools that only remove metadata or compress embedded objects, this approach directly reduces page resolution, allowing significant file size reduction.

---

## What is DPI?

**DPI (Dots Per Inch)** is a measurement of image resolution.

When a PDF page is converted into an image, DPI determines how many pixels are used to represent that page.

### Simple Example

Think of DPI as the amount of detail captured in a photograph.

```text
Higher DPI
    ↓
More Pixels
    ↓
Sharper Quality
    ↓
Larger File Size
```

```text
Lower DPI
    ↓
Fewer Pixels
    ↓
Less Detail
    ↓
Smaller File Size
```

---

## DPI Levels Used by the System

The compressor tries multiple DPI values:

| DPI | Quality  | Typical Result                |
| --- | -------- | ----------------------------- |
| 150 | High     | Sharp text and images         |
| 96  | Medium   | Good readability              |
| 72  | Standard | Acceptable for most documents |
| 48  | Low      | Noticeable quality reduction  |
| 32  | Very Low | Maximum compression           |

The process starts with higher quality and gradually lowers the DPI until the requested file size is achieved.

---

## Why DPI Affects File Size

A PDF page is first rendered as an image.

For an A4 page:

| DPI | Approximate Resolution |
| --- | ---------------------- |
| 150 | 1240 × 1754 px         |
| 96  | 794 × 1123 px          |
| 72  | 595 × 842 px           |
| 48  | 397 × 561 px           |
| 32  | 265 × 374 px           |

As DPI decreases:

* Fewer pixels are generated.
* JPEG images become smaller.
* Rebuilt PDFs become smaller.

This is the primary mechanism used to achieve target file sizes.

---

## Compression Workflow

### Step 1: Open PDF

The uploaded PDF is loaded using PyMuPDF.

```python
doc = fitz.open(input_path)
```

---

### Step 2: Select Target Size

Available compression targets:

| Option | Target Size    |
| ------ | -------------- |
| 50     | 50 KB          |
| 100    | 100 KB         |
| 500    | 500 KB         |
| 0      | No size target |

If no target is selected, the PDF is optimized without reducing page quality.

```python
doc.save(
    output_path,
    garbage=4,
    deflate=True,
    clean=True
)
```

This removes unused objects and compresses internal PDF structures.

---

### Step 3: Render Each Page as an Image

Each page is rasterized using the selected DPI.

```python
mat = fitz.Matrix(dpi / 72, dpi / 72)
pix = page.get_pixmap(matrix=mat)
```

Workflow:

```text
PDF Page
    ↓
Rendered Image
```

---

### Step 4: JPEG Compression

The rendered image is converted into JPEG format.

```python
img_bytes = pix.tobytes("jpeg")
```

JPEG compression significantly reduces image size while maintaining visual readability.

```text
Rendered Image
      ↓
JPEG Compression
      ↓
Smaller Image Data
```

---

### Step 5: Rebuild PDF

A new PDF is created and compressed images are inserted as pages.

```python
new_page.insert_image(
    img_rect,
    stream=img_bytes
)
```

Workflow:

```text
Original PDF
      ↓
Render Pages
      ↓
JPEG Compression
      ↓
Create New PDF
```

---

### Step 6: Check File Size

After rebuilding the PDF:

```python
size = os.path.getsize(tmp)
```

The generated size is compared against the user's selected target.

```python
if size <= target_bytes:
```

If successful:

```text
Target Reached
      ↓
Save PDF
```

Otherwise:

```text
Still Too Large
      ↓
Reduce DPI
      ↓
Try Again
```

---

### Step 7: Progressive DPI Reduction

The compressor automatically tests:

```text
150 DPI
   ↓
96 DPI
   ↓
72 DPI
   ↓
48 DPI
   ↓
32 DPI
```

Compression stops when:

* Target size is achieved.
* Minimum DPI (32) is reached.

---

## Example Compression Scenario

Suppose:

```text
Original PDF = 3.2 MB
Target Size = 100 KB
```

Compression attempts:

| Attempt | DPI | Result   |
| ------- | --- | -------- |
| 1       | 150 | 420 KB ❌ |
| 2       | 96  | 240 KB ❌ |
| 3       | 72  | 145 KB ❌ |
| 4       | 48  | 98 KB ✅  |

Since 98 KB is below the 100 KB target, the 48 DPI version is returned.

---

## Trade-Offs

This method achieves very high compression ratios but changes the PDF structure.

### Advantages

* Significant size reduction
* Predictable output size
* Works with almost any PDF
* Fast processing
* Suitable for sharing and storage

### Limitations

Since pages become images:

* Text is no longer selectable.
* Search functionality is removed.
* Embedded fonts are lost.
* Forms become non-editable.
* Quality decreases at lower DPI values.

The final output is an image-based PDF rather than a text-based PDF.

---

## Complete Compression Flow

```text
Upload PDF
    ↓
Open PDF
    ↓
Render Pages at 150 DPI
    ↓
Convert Pages to JPEG
    ↓
Rebuild PDF
    ↓
Check File Size

Target Achieved?
    ├── Yes → Save PDF
    └── No
            ↓
      Lower DPI
            ↓
        Repeat

32 DPI Reached?
    ├── Yes → Save Best Result
    └── No → Continue
```

This progressive DPI reduction strategy is the core mechanism used by the application to compress PDFs while attempting to balance quality and file size.
