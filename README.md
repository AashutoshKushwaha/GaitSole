# GaitSole

**Regional foot-sole force simulation & low-latency gait prediction. This is part of a larger project to develop and simulate HoloTile -a Virtual Reality Rehabilitation System as well as for training of bots and vehicles for unexplored realistic terrains.**

GaitSole is a two-part biomechanics project:

1. **`simulation_scripts/` — an OpenSim + Moco pipeline** that simulates 2D and 3D
   running/walking gait and reads **per-region foot-sole ground-reaction
   force** (heel / midfoot / forefoot / toe per foot) instead of the usual
   single whole-foot GRF. Each contact region acts like an independent
   load-cell.
2. **`motion_predictor/` — a lightweight PyTorch model** that observes a short
   history of lower-limb kinematics and predicts, at sub-millisecond latency,
   the next pose plus per-foot force/moment. It learns from the kind of data
   the OpenSim pipeline produces.

Together they go from *physics-based gait simulation with detailed plantar
loading* → *a fast learned predictor of motion and foot loading*.

---

## Why this exists

Standard running-biomechanics models report one GRF vector per foot. Many
questions in footwear design, injury research, and orthotics depend on **where
under the foot** the load sits (heel strike vs. forefoot push-off). GaitSole's
4-region contact model exposes that spatial breakdown directly from a
dynamically-consistent simulation, and the ML predictor makes those quantities
available in real time without re-solving an expensive optimal-control problem.

---

## Repository structure

```
GaitSole/
├── README.md                      # this file
├── simulation_scripts/                       # OpenSim + Moco simulation pipeline (run in order)
│   ├── 01_inspect_2d_gait.py            # inspect the shipped 2D gait model
│   ├── 02_forward_sim_per_region_forces.py  # forward sim, read per-region force
│   ├── 03_upgrade_to_4_regions.py       # build 2D_gait_4regions.osim (heel/mid/fore/toe)
│   ├── 04_read_force_sto.py             # read & plot per-region force .sto
│   ├── 05_predictive_running_moco.py    # 2D predictive running (Moco optimal control)
│   ├── 06_extract_grf_from_solution.py  # per-region GRF from a saved solution
│   ├── 07_walk_driven_per_region_grf.py # per-region GRF driven by reference walking
│   ├── 08_tile_strides.py               # tile the periodic cycle into N strides
│   ├── 09_build_rajagopal_arms_fingers.py  # build 3D Rajagopal2016 + 4-region feet + fingers
│   ├── 10_inspect_and_test_rajagopal.py # inspect + prescribed-motion preview of 3D model
│   └── 11_predictive_running_3d_moco.py # 3D predictive running (the real, slow solve)
│
└── motion_predictor/              # PyTorch low-latency predictor
    ├── config.py                  # all variables, dims, hyperparameters, paths
    ├── data.py                    # synthetic gait, featurization, windowing, normalization
    ├── model.py                   # MotionPredictor (MLP trunk + pose/rootvel/kinetics heads)
    ├── train.py                   # training loop (synthetic by default; --data for real)
    ├── infer_stream.py            # real-time streaming inference + latency benchmark
    ├── requirements.txt           # torch + numpy
    └── README.md                  # predictor-specific details
```

> **Models are generated, not committed.** Scripts `03` and `09` *build* the
> `.osim` models from base OpenSim assets, so the repo ships only code. The
> simulation `.osim`/`.sto` outputs land in an `output/` folder created at run
> time (see each script's header for exact paths).

---

## The simulation pipeline at a glance

| Step | Script | What it does |
|------|--------|--------------|
| 1 | `01_inspect_2d_gait.py` | Enumerate joints/coordinates/muscles/contacts of the shipped `2D_gait.osim`. |
| 2 | `02_forward_sim_per_region_forces.py` | Forward-simulate and read each contact sphere's force vector (per-region load-cell). |
| 3 | `03_upgrade_to_4_regions.py` | Produce `2D_gait_4regions.osim` — splits the front contact into midfoot + forefoot + toe. |
| 4 | `04_read_force_sto.py` | Load the ForceReporter `.sto`, plot per-region vertical force. |
| 5 | `05_predictive_running_moco.py` | Moco predictive **2D running** half-stride with anti-symmetric periodicity (~10–30 min solve). |
| 6 | `06_extract_grf_from_solution.py` | Replay a saved solution to extract per-region GRF without re-solving. |
| 7 | `07_walk_driven_per_region_grf.py` | Drive the 4-region model with reference **walking** kinematics → per-region GRF (fast, reliable). |
| 8 | `08_tile_strides.py` | Mirror the half-stride to a full stride and tile into N strides. |
| 9 | `09_build_rajagopal_arms_fingers.py` | Build the **3D** `Rajagopal2016_4regions_fingers.osim` (4-region feet + articulated fingers). |
| 10 | `10_inspect_and_test_rajagopal.py` | Inspect the 3D model + generate a prescribed-motion preview. |
| 11 | `11_predictive_running_3d_moco.py` | Moco predictive **3D running** — the full physics solve (long runtime). |

---

## Prerequisites

### For the simulation pipeline (`scripts/`)
- **OpenSim 4.5** with the **Python bindings** and **Moco** (Moco ships inside
  OpenSim 4.5). The pipeline was developed against the OpenSim 4.5 install at
  `E:/OpenSim/4.5/...`; adjust the hard-coded `MODEL`/path constants at the top
  of each script to match your install.
- The shipped **`2D_gait.osim`** (from `…/Moco/example2DWalking/`) — used by
  steps 1–3.
- The canonical **`Rajagopal2016.osim`** — used by step 9.
- Python packages: `numpy`, `pandas`, `matplotlib` (for reading/plotting `.sto`).

> ⚠️ The OpenSim Python bindings only import from the OpenSim-provided Python
> interpreter (a dedicated conda env), **not** a generic system Python. Run the
> `scripts/` files with that interpreter — e.g.
> `E:/conda/envs/opensim_env/python.exe`.

### For the predictor (`motion_predictor/`)
- Python 3.10+ with **PyTorch ≥ 2.0** and **NumPy ≥ 1.23**.
- Runs on plain CPU for the synthetic demo; a GPU helps for real-data training.

---

## Installation

```bash
# 1. Clone
git clone https://github.com/<you>/GaitSole.git
cd GaitSole
```

### OpenSim environment (for scripts/)
Install OpenSim 4.5 (with Moco) and its Python bindings. The most reliable
route is the conda package:

```bash
conda create -n opensim_env python=3.10
conda activate opensim_env
conda install -c opensim-org opensim          # provides opensim + Moco bindings
pip install numpy pandas matplotlib
```

Confirm the bindings import:

```bash
python -c "import opensim; print(opensim.GetVersion())"
```

### Predictor environment (for motion_predictor/)
```bash
cd motion_predictor
pip install -r requirements.txt                # torch + numpy
```

---

## Running the simulation pipeline

The scripts are numbered and meant to run **in order**. Use the OpenSim Python
interpreter and edit the path constants at the top of each file to point at
your OpenSim 4.5 install before the first run.

```bash
conda activate opensim_env
cd scripts

# 2D track
python 01_inspect_2d_gait.py
python 02_forward_sim_per_region_forces.py            # default: shipped 2-sphere model
python 03_upgrade_to_4_regions.py                     # writes 2D_gait_4regions.osim
python 02_forward_sim_per_region_forces.py 4regions   # re-run on the 4-region model
python 04_read_force_sto.py                           # plot per-region force

python 05_predictive_running_moco.py                  # SLOW: ~10–30 min predictive solve
python 06_extract_grf_from_solution.py                # per-region GRF from the saved solution
python 07_walk_driven_per_region_grf.py               # fast walking-driven GRF
python 08_tile_strides.py 5                            # tile into 5 strides

# 3D track
python 09_build_rajagopal_arms_fingers.py             # writes Rajagopal2016_4regions_fingers.osim
python 10_inspect_and_test_rajagopal.py               # inspect + prescribed-motion preview
python 11_predictive_running_3d_moco.py               # SLOW: full 3D physics solve
```

**Outputs** (kinematics and per-region force `.sto` files) are written under
`E:/OpenSim/output/` by default; load the `.osim` + `.sto` pairs in the OpenSim
GUI to visualize. Each script's docstring lists its exact inputs and outputs.

> 💡 Steps 5 and 11 are optimal-control solves and can take many minutes (11 is
> the slow one). Steps 6 and 8 deliberately operate on a *saved* solution so a
> post-processing bug never forces a re-solve.

## Running the motion predictor

No download or OpenSim needed for the synthetic demo — it validates the full
machinery end-to-end:

```bash
cd motion_predictor
python train.py            # trains on synthetic gait → runs/predictor.pt
python infer_stream.py     # streaming inference + latency benchmark
```

Reference result on a laptop CPU: **~0.9 ms/prediction (~1100 fps)**. To train
on real data, implement `load_real(path)` in `train.py` and run
`python train.py --data <dir_of_trials> --device cuda --epochs 100`. See
`motion_predictor/README.md` for the data format and recommended public
datasets (Schreiber & Moissenet 2019; Camargo et al. 2021).

---

## What the predictor predicts

Per-frame input is a 26-number lower-limb feature vector; outputs are split into:

- **next pose** — lower-limb joint-angle residuals + pelvis orientation/height
  + root horizontal velocity (12 + 2),
- **foot kinetics** — per-foot ground-reaction force `Fx, Fy, Fz`, free moment
  `Mz`, and centre of pressure `COPx, COPz` (12).

Design choices (joint angles not 3D points; root as velocity not absolute
position; predict residuals then integrate) are explained in
`motion_predictor/README.md`.

---

## Roadmap

- **Per-region foot force in the predictor** (heel/midfoot/forefoot/toe) trained
  on the OpenSim 4-region contact data — closing the loop between the two halves
  of GaitSole.
- Whole-body linear/angular momentum output head.
- Configurable longer prediction horizon.
- Live input source (video / IMU / mocap stream) feeding `infer_stream.py`.

---

## Notes & caveats

- Paths in `scripts/` are currently **hard-coded to a `E:/OpenSim/...` layout**.
  Update the `MODEL`/`OUT`/`REF` constants at the top of each script for your
  machine, or refactor them into a shared config — a good first contribution.
- The synthetic data in `motion_predictor/` only proves the pipeline works; real
  accuracy requires real trials.
- Finger articulation in the 3D model is simplified to pure flex/extend hinges
  (see `09_build_rajagopal_arms_fingers.py` header for the anatomical
  assumptions).

---

## License

This project is licensed under the [MIT License](LICENSE).
```

