# Hi-SuM Dataset - Data Navigation Guide (DANDI:001510)

**Whole-Cell Patch-Clamp and Fluorescence Imaging of SuM Neurons in Mouse Brain Slices**
DANDI dandiset **001510** - https://dandiarchive.org/dandiset/001510

Two modalities, one NWB file each, organized by subject. Each file is
**self-describing**: the recording data plus the metadata needed to interpret it
(which channels belong to which cell, which image this is) travels inside the
file. The complete source spreadsheets are included verbatim under `sourcedata/`.

---

## 1. What's in the dataset

| Modality | Suffix | Subjects | One NWB = |
| --- | --- | --- | --- |
| **Slice electrophysiology** | `_icephys.nwb` | 49 subjects (LG282-LG639) | one recording **session** (all cells + files that day) |
| **Confocal puncta imaging** | `_ophys.nwb` | 16 subjects (LG445-LG673) | one **image** (ND2 z-stack); 10 images/subject |

### Layout
```
dandiset.yaml
dataset_description.json   .bidsignore      # BIDS markers (let DANDI host sourcedata/)
DATA_GUIDE.md
sourcedata/                                 # the complete source spreadsheets, verbatim
    file_organization.xlsx                      slice-ephys master index
    Summarized_puncta_data_nd2_read.xlsx
    Summarized_puncta_data_nd2_read_added_puncta.xlsx
sub-<ID>/
    sub-<ID>_ses-<YYYYMMDD>_icephys.nwb
    sub-<ID>_ses-<YYYYMMDD>_obj-<hash>_ophys.nwb
```

Filenames tell you subject + date directly (`sub-LG282_ses-20221216_icephys.nwb`
= subject LG282, recorded 2022-12-16). Subject **sex, strain (mouse line), genotype
and age** show on the dandiset landing page (DANDI derives them from each NWB's
`Subject`); **injection location, virus and sacrifice details** live in the `Subject`
description/notes (and, for imaging, in `image_roi_info`).

---

## 2. Slice electrophysiology (`_icephys.nwb`)

### Which channels belong to which cell - read it inside the file
Each icephys NWB embeds a **`metadata` processing module** built from
`file_organization.xlsx`, with two tables:

- **`cell_channel_map`** - one row per **(cell x ABF file x channel)**:

  | column | meaning |
  | --- | --- |
  | `cell_code` | the cell (e.g. `LG282_1_2nf_CH1`; `CH1`/`CH2` = amplifier headstage) |
  | `abf_file`, `file_type` | which raw ABF and its protocol type |
  | `channel`, `unit` | ADC channel + unit (pA / mV / TTL) |
  | `role` | **`data`** = this cell's recording; **`sync`** = shared light-delivery command TTL |
  | `nwb_series` | the matching series in `/acquisition` |
  | `present_in_nwb` | that series is actually in this file |

  So to see a cell's channels, filter this table by `cell_code` and look at
  `role = data`. In a light-pulse file a cell typically has **two data channels**
  (one pA voltage-clamp current + one mV current-clamp voltage) plus the shared
  `IN 6` TTL (`role = sync`).

- **`session_source_records`** - the raw `file_organization.xlsx` rows for this
  session, verbatim (CellType, LightResponse, InjectionScheme, etc.).

The same information is also written in prose on each **`IntracellularElectrode`**
description (`general` > `intracellular_ephys` > `<cell_code>`), grouped by file, e.g.
"Data channels by file - 2022_12_16_0006.abf: Imemb (pA), Vm_sec1 (mV); ... Command TTL for light delivery: IN 6 (TTL) (on/off command pulse, common to all cells recorded on this file)."

### The traces
`acquisition` holds the sweeps, typed by the file's **clamp mode** (from `FileType`),
not by channel unit - all channels in a file share one clamp mode:
- a **current-voltage recording** (current clamp) -> `CurrentClampSeries` (recorded
  voltage, volts) + `CurrentClampStimulusSeries` (command current, amperes);
- a **light-pulse** file (voltage clamp) -> `VoltageClampSeries` (recorded current,
  amperes) + `VoltageClampStimulusSeries` (command voltage, volts).

The shared light-delivery command TTL is a `TimeSeries` named `<abf>_<chan>_TTL`
(written once per ABF, common to all cells on a file). Each series links to its
`electrode` (= which cell). Two cells can share one ABF (patched on CH1 + CH2) -
each resolves to its own channels in `cell_channel_map`.

---

## 3. Confocal puncta imaging (`_ophys.nwb`)

Each ophys NWB is **one ND2 z-stack = one ROI**, and it is self-describing.

### Which ROI / section is this image - read it inside the file
Each ophys NWB embeds a **`metadata` processing module** with one table:

- **`image_roi_info`** - this image's identity + its cleaned-puncta count, from
  the puncta workbooks (one row per quantification method):

  | column | meaning |
  | --- | --- |
  | `roi_label` (`ROI_1..10`), `roi_index` (1-10) | which ROI this image is (**1-5 anterior, 6-10 posterior**) |
  | `roi_source_filename_column` (`Filename_i..v`) | the spreadsheet column it maps to |
  | `mouse_id`, `mouse_line`, `sex`, `injection_location`, `virus`, `anterior_posterior` | section + subject identity |
  | `quant_method` | `automated` or `manual` (the two source workbooks) |
  | `puncta_count_cleaned` | this ROI's cleaned-puncta count (the only count published) |
  | `source_excel`, `nd2_file`, `roi_source_filename` | which workbook/row and the exact ND2 image + filename cell |
  | `roi_1_ipsilateral`, `roi_5_ipsilateral`, `threshold_used`, `date_captured` | extra section identity / analysis provenance |

  **`ROI_N` is the image in `Filename_N`** (positional; the raw file numbering is
  reversed - `Filename_i` is the `...EGFP-5` file). Each image appears **twice** -
  once for the automated count and once for the manual count (`quant_method`).

The ND2 file stem is also in `session_description` (the join key to the workbook),
and `acquisition` holds one `OnePhotonSeries` per channel (DAPI, EGFP, and the
thresholded/puncta masks).

---

## 4. The source spreadsheets (`sourcedata/`)

The full source data - every animal, ROI, and analysis - verbatim `.xlsx`:

| Workbook / tab | Contents |
| --- | --- |
| `file_organization.xlsx` -> `Sheet1` | slice-ephys master index (all ABF files, `Channels - Units`, cell types) |
| `Summarized_puncta_data_nd2_read.xlsx` -> `Automated quantification only` | per-section x 5-ROI **automated** cleaned-puncta counts, with `Filename_i..v` |
| `..._added_puncta.xlsx` -> `Manual data` | the same sections/ROIs with **manually** counted cleaned puncta |

---

## 5. How to open

### In the browser - Neurosift
Dandiset **Files** > open a subject folder > click a `.nwb` > **Open with Neurosift**.
- **icephys:** expand **`acquisition`** and click a series to plot the trace;
  expand **`processing` > `metadata` > `cell_channel_map`** to see the cell-to-channel
  table; expand **`general` > `intracellular_ephys` > `<cell>`** for the electrode
  description.
- **ophys:** expand **`acquisition`** and click a `OnePhotonSeries` to view the
  image; expand **`processing` > `metadata` > `image_roi_info`** for its ROI /
  section identity and puncta counts.

### Locally with pynwb
```python
from pynwb import NWBHDF5IO
with NWBHDF5IO("sub-LG282_ses-20221216_icephys.nwb", "r") as io:
    nwb = io.read()
    print(nwb.processing["metadata"]["cell_channel_map"].to_dataframe())   # channels per cell
    s = nwb.acquisition[next(iter(nwb.acquisition))]
    d = s.data[:] * s.conversion                     # volts (CC) or amperes (VC)
    print(s.name, d.min(), d.max(), len(d))          # NWB stores no min/max - compute it
```

---

## 6. Quick reference

| I want to know... | Look at... |
| --- | --- |
| Subject sex/DOB/injection/virus/sacrifice | the NWB `Subject` (also on the landing page) |
| **Which channels are a cell's data vs sync** | that NWB's `processing/metadata/cell_channel_map` (filter `cell_code`, `role`) |
| Full sheet context for a session | that NWB's `processing/metadata/session_source_records`, or `sourcedata/file_organization.xlsx` |
| Which ROI/section an ophys NWB is | that NWB's `processing/metadata/image_roi_info` (or `session_description` for the ND2 stem) |
| Puncta counts for an image | that NWB's `processing/metadata/image_roi_info`, or the `sourcedata/...` puncta workbooks by section + ROI (`ROI_N` <-> `Filename_N`) |
| Automated vs manual puncta counts | that NWB's `image_roi_info` (`quant_method`), or the two `sourcedata/` puncta workbooks |
| Sample min/max/mean of a trace | compute from `series.data` - NWB does not store it |
