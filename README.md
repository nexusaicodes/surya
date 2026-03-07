# Surya Tabular OCR

A trimmed fork of [Surya](https://github.com/VikParuchuri/surya) focused on **English-only OCR, layout analysis, and table recognition** from document images.

|                            Detection                             |                                   OCR                                   |
|:----------------------------------------------------------------:|:-----------------------------------------------------------------------:|
|  <img src="static/images/excerpt.png" width="500px"/>  |  <img src="static/images/excerpt_text.png" width="500px"/> |

|                               Layout                               |                       Table Recognition                       |
|:------------------------------------------------------------------:|:-------------------------------------------------------------:|
| <img src="static/images/excerpt_layout.png" width="500px"/> | <img src="static/images/scanned_tablerec.png" width="500px"/> |

## What's included

- **OCR** — detect text lines and read text from document images
- **Layout analysis** — detect tables, headers, sections, figures, etc. with reading order
- **Table recognition** — extract rows, columns, and cells from tables

## What's removed (vs upstream Surya)

- Multilingual support (90+ languages reduced to English only)
- LaTeX/math OCR (texify)
- Reading order as standalone feature
- Streamlit interactive UIs
- OCR error detection

# Installation

Requires Python 3.10+ and PyTorch. You may need to install the CPU version of torch first if you're not using a Mac or a GPU machine. See [here](https://pytorch.org/get-started/locally/) for more details.

```shell
pip install surya-tabular-ocr
```

Model weights will automatically download the first time you run surya.

# Usage

- Inspect the settings in `surya/settings.py`. You can override any setting with environment variables.
- Your torch device will be automatically detected, but you can override this. For example, `TORCH_DEVICE=cuda`.

## OCR (text recognition)

This command will write out a json file with the detected text and bboxes:

```shell
surya_ocr DATA_PATH
```

- `DATA_PATH` can be an image, pdf, or folder of images/pdfs
- `--task_name` will specify which task to use for predicting the lines. `ocr_with_boxes` is the default, which will format text and give you bboxes. If you get bad performance, try `ocr_without_boxes`, which will give you potentially better performance but no bboxes. For blocks like equations and paragraphs, try `block_without_boxes`.
- `--images` will save images of the pages and detected text lines (optional)
- `--output_dir` specifies the directory to save results to instead of the default
- `--page_range` specifies the page range to process in the PDF, specified as a single number, a comma separated list, a range, or comma separated ranges - example: `0,5-10,20`.

The `results.json` file will contain a json dictionary where the keys are the input filenames without extensions. Each value will be a list of dictionaries, one per page of the input document. Each page dictionary contains:

- `text_lines` - the detected text and bounding boxes for each line
  - `text` - the text in the line
  - `confidence` - the confidence of the model in the detected text (0-1)
  - `polygon` - the polygon for the text line in (x1, y1), (x2, y2), (x3, y3), (x4, y4) format. The points are in clockwise order from the top left.
  - `bbox` - the axis-aligned rectangle for the text line in (x1, y1, x2, y2) format. (x1, y1) is the top left corner, and (x2, y2) is the bottom right corner.
  - `chars` - the individual characters in the line
    - `text` - the text of the character
    - `bbox` - the character bbox (same format as line bbox)
    - `polygon` - the character polygon (same format as line polygon)
    - `confidence` - the confidence of the model in the detected character (0-1)
    - `bbox_valid` - if the character is a special token, the bbox may not be valid
  - `words` - the individual words in the line (computed from the characters)
    - `text` - the text of the word
    - `bbox` - the word bbox (same format as line bbox)
    - `polygon` - the word polygon (same format as line polygon)
    - `confidence` - mean character confidence
    - `bbox_valid` - if the word is a special token, the bbox may not be valid
- `page` - the page number in the file
- `image_bbox` - the bbox for the image in (x1, y1, x2, y2) format. (x1, y1) is the top left corner, and (x2, y2) is the bottom right corner. All line bboxes will be contained within this bbox.

**Performance tips**

Setting the `RECOGNITION_BATCH_SIZE` env var properly will make a big difference when using a GPU. Each batch item will use `40MB` of VRAM, so very high batch sizes are possible. The default is a batch size `512`, which will use about 20GB of VRAM. Depending on your CPU core count, it may help, too - the default CPU batch size is `32`.

### From python

```python
from PIL import Image
from surya.foundation import FoundationPredictor
from surya.recognition import RecognitionPredictor
from surya.detection import DetectionPredictor

image = Image.open(IMAGE_PATH)
foundation_predictor = FoundationPredictor()
recognition_predictor = RecognitionPredictor(foundation_predictor)
detection_predictor = DetectionPredictor()

predictions = recognition_predictor([image], det_predictor=detection_predictor)
```

## Layout analysis

This command will write out a json file with the detected layout and reading order.

```shell
surya_layout DATA_PATH
```

- `DATA_PATH` can be an image, pdf, or folder of images/pdfs
- `--images` will save images of the pages and detected text lines (optional)
- `--output_dir` specifies the directory to save results to instead of the default
- `--page_range` specifies the page range to process in the PDF, specified as a single number, a comma separated list, a range, or comma separated ranges - example: `0,5-10,20`.

The `results.json` file will contain a json dictionary where the keys are the input filenames without extensions. Each value will be a list of dictionaries, one per page of the input document. Each page dictionary contains:

- `bboxes` - detected bounding boxes for text
  - `bbox` - the axis-aligned rectangle for the text line in (x1, y1, x2, y2) format. (x1, y1) is the top left corner, and (x2, y2) is the bottom right corner.
  - `polygon` - the polygon for the text line in (x1, y1), (x2, y2), (x3, y3), (x4, y4) format. The points are in clockwise order from the top left.
  - `position` - the reading order of the box.
  - `label` - the label for the bbox. One of `Caption`, `Footnote`, `Formula`, `ListItem`, `PageFooter`, `PageHeader`, `Picture`, `Figure`, `SectionHeader`, `Table`, `Form`, `TableOfContents`, `Text`, `Code`, `Equation`.
  - `top_k` - the top-k other potential labels for the box. A dictionary with labels as keys and confidences as values.
- `page` - the page number in the file
- `image_bbox` - the bbox for the image in (x1, y1, x2, y2) format. (x1, y1) is the top left corner, and (x2, y2) is the bottom right corner. All line bboxes will be contained within this bbox.

**Performance tips**

Setting the `LAYOUT_BATCH_SIZE` env var properly will make a big difference when using a GPU. Each batch item will use `220MB` of VRAM, so very high batch sizes are possible. The default is a batch size `32`, which will use about 7GB of VRAM. Depending on your CPU core count, it might help, too - the default CPU batch size is `4`.

### From python

```python
from PIL import Image
from surya.foundation import FoundationPredictor
from surya.layout import LayoutPredictor
from surya.settings import settings

image = Image.open(IMAGE_PATH)
layout_predictor = LayoutPredictor(FoundationPredictor(checkpoint=settings.LAYOUT_MODEL_CHECKPOINT))

# layout_predictions is a list of dicts, one per image
layout_predictions = layout_predictor([image])
```

## Table Recognition

This command will write out a json file with the detected table cells and row/column ids, along with row/column bounding boxes.

```shell
surya_table DATA_PATH
```

- `DATA_PATH` can be an image, pdf, or folder of images/pdfs
- `--images` will save images of the pages and detected table cells + rows and columns (optional)
- `--output_dir` specifies the directory to save results to instead of the default
- `--page_range` specifies the page range to process in the PDF, specified as a single number, a comma separated list, a range, or comma separated ranges - example: `0,5-10,20`.
- `--detect_boxes` specifies if cells should be detected. By default, they're pulled out of the PDF, but this is not always possible.
- `--skip_table_detection` tells table recognition not to detect tables first. Use this if your image is already cropped to a table.

The `results.json` file will contain a json dictionary where the keys are the input filenames without extensions. Each value will be a list of dictionaries, one per page of the input document. Each page dictionary contains:

- `rows` - detected table rows
  - `bbox` - the bounding box of the table row
  - `row_id` - the id of the row
  - `is_header` - if it is a header row.
- `cols` - detected table columns
  - `bbox` - the bounding box of the table column
  - `col_id`- the id of the column
  - `is_header` - if it is a header column
- `cells` - detected table cells
  - `bbox` - the axis-aligned rectangle for the text line in (x1, y1, x2, y2) format. (x1, y1) is the top left corner, and (x2, y2) is the bottom right corner.
  - `text` - if text could be pulled out of the pdf, the text of this cell.
  - `row_id` - the id of the row the cell belongs to.
  - `col_id` - the id of the column the cell belongs to.
  - `colspan` - the number of columns spanned by the cell.
  - `rowspan` - the number of rows spanned by the cell.
  - `is_header` - whether it is a header cell.
- `page` - the page number in the file
- `table_idx` - the index of the table on the page (sorted in vertical order)
- `image_bbox` - the bbox for the image in (x1, y1, x2, y2) format. (x1, y1) is the top left corner, and (x2, y2) is the bottom right corner. All line bboxes will be contained within this bbox.

**Performance tips**

Setting the `TABLE_REC_BATCH_SIZE` env var properly will make a big difference when using a GPU. Each batch item will use `150MB` of VRAM, so very high batch sizes are possible. The default is a batch size `64`, which will use about 10GB of VRAM. Depending on your CPU core count, it might help, too - the default CPU batch size is `8`.

### From python

```python
from PIL import Image
from surya.table_rec import TableRecPredictor

image = Image.open(IMAGE_PATH)
table_rec_predictor = TableRecPredictor()

table_predictions = table_rec_predictor([image])
```

## SDK Pipeline

For a single-call interface that runs layout detection, table recognition, and OCR together:

```python
from surya.pipeline import TableExtractionPipeline

pipeline = TableExtractionPipeline()  # loads all models once
result = pipeline.extract_tables(pil_image, ocr=True)
# result is a plain dict, JSON-serializable
# {"tables": [...], "image_size": [w, h]}
```

- Accepts `PIL.Image` or raw `bytes`
- `ocr=True` runs text recognition on each detected table
- `skip_table_detection=True` treats the whole image as one table

## Compilation

The following models support torch compilation. Set environment variables to enable:

- Detection: `COMPILE_DETECTOR=true`
- Layout: `COMPILE_LAYOUT=true`
- Table recognition: `COMPILE_TABLE_REC=true`

Or set `COMPILE_ALL=true` to compile all models.

# Limitations

- Specialized for document OCR. It will likely not work on photos or other images.
- For printed text, not handwriting (though it may work on some handwriting).
- The text detection model has trained itself to ignore advertisements.

## Troubleshooting

If OCR isn't working properly:

- Try increasing resolution of the image so the text is bigger. If the resolution is already very high, try decreasing it to no more than a `2048px` width.
- Preprocessing the image (binarizing, deskewing, etc) can help with very old/blurry images.
- You can adjust `DETECTOR_BLANK_THRESHOLD` and `DETECTOR_TEXT_THRESHOLD` if you don't get good results. `DETECTOR_BLANK_THRESHOLD` controls the space between lines - any prediction below this number will be considered blank space. `DETECTOR_TEXT_THRESHOLD` controls how text is joined - any number above this is considered text. `DETECTOR_TEXT_THRESHOLD` should always be higher than `DETECTOR_BLANK_THRESHOLD`, and both should be in the 0-1 range. Looking at the heatmap from the debug output of the detector can tell you how to adjust these.

# Development

```bash
git clone https://github.com/sakshamsaxena/surya.git
cd surya
uv sync --group dev    # install main + dev dependencies
uv run pytest          # run tests
```

## Running benchmarks

```bash
uv run python benchmark/detection.py --max_rows 256
uv run python benchmark/recognition.py
uv run python benchmark/layout.py
uv run python benchmark/table_recognition.py --max_rows 1024
```

All benchmark scripts accept `--max_rows`, `--debug`, and `--results_dir` flags.

# Finetuning

You can finetune Surya OCR on your own data with the [finetuning script](/surya/scripts/finetune_ocr.py). It's built on Hugging Face Trainer, and supports all the [arguments](https://huggingface.co/docs/transformers/en/main_classes/trainer#transformers.TrainingArguments) that the huggingface trainer provides.

```bash
# Tested on 1xH100 GPU
python surya/scripts/finetune_ocr.py \
  --output_dir $OUTPUT_DIR \
  --dataset_name datalab-to/ocr_finetune_example \
  --per_device_train_batch_size 64 \
  --gradient_checkpointing true \
  --max_sequence_length 1024
```

# License

Code is licensed under GPL-3.0-or-later. Model weights use a modified AI Pubs Open Rail-M license (free for research, personal use, and startups under $2M funding/revenue). See [LICENSE](LICENSE) and [MODEL_LICENSE](MODEL_LICENSE) for details.

# Acknowledgments

This is a trimmed fork of [Surya](https://github.com/VikParuchuri/surya) by Vik Paruchuri and the Datalab team. The original work builds on [EfficientViT](https://github.com/mit-han-lab/efficientvit), [Donut](https://github.com/clovaai/donut), [transformers](https://github.com/huggingface/transformers), and others.
