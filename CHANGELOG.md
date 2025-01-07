### [v0.1.7] - [10/2024]

##### Added

- Adds an option for device-parallel evaluation of `BBOBFitness`.
- Implements fully `pmap`-compatible implementations of `OpenES`, `PGPE`, `Sep_CMA_ES` and `SNES`. Example: [`09_pmap_strategy.ipynb`](https://github.com/RobertTLange/evosax/blob/main/examples/09_pmap_strategy.ipynb):

```python
# set number of cpu devices for jax pmap
import os

num_devices = 4
os.environ["XLA_FLAGS"] = f"--xla_force_host_platform_device_count={num_devices}"

import jax
import jax.numpy as jnp
from flax import jax_utils

print(jax.devices())

from evosax.problems import BBOBFitness
from evosax.v2 import SNES

fn_name = "Sphere"
num_dims = 2
popsize = 64
rng = jax.random.PRNGKey(0)
evaluator = BBOBFitness(fn_name, num_dims=num_dims, n_devices=num_devices)

strategy = SNES(
    popsize=popsize,
    num_dims=num_dims,
    sigma_init=0.1,
    n_devices=num_devices,
    maximize=False,
)

es_params = strategy.default_params.replace(init_min=-3.0, init_max=3.0)
es_params = jax_utils.replicate(es_params)

init_rng = jnp.tile(rng[None], (num_devices, 1))
es_state = jax.pmap(strategy.initialize)(init_rng, es_params)
print("Mean pre-update:", es_state.mean)  # (num_devices, num_dims)

rng, rng_a, rng_e = jax.random.split(rng, 3)
ask_rng = jax.random.split(rng_a, num_devices)
x, es_state = jax.pmap(strategy.ask, axis_name="device")(ask_rng, es_state, es_params)

print("Population shape:", x.shape)  # (num_devices, popsize/num_devices, num_dims)

fitness = evaluator.rollout(rng_e, x)
print("Fitness shape:", fitness.shape)  # (num_devices, popsize/num_devices)

es_state = jax.pmap(strategy.tell, axis_name="device")(x, fitness, es_state, es_params)
print("Mean post-update:", es_state.mean)  # (num_devices, num_dims)
```

- Added `DiffusionEvolution` based on [Zhang et al. (2024)](https://arxiv.org/pdf/2410.02543). Example: [`10_diffusion_evolution.ipynb`](https://github.com/RobertTLange/evosax/blob/main/examples/10_diffusion_evolution.ipynb)

- Added `SV_CMA_ES` ([Braun et al., 2024](https://arxiv.org/abs/2410.10390)) and `SV_OpenES` ([Liu et al., 2017](https://arxiv.org/abs/1704.02399)) as subpopulation ES with stein variational updates.

Big thanks to Cornelius Braun (@cornelius-braun
) for adding the stein variational methods!

### [v0.1.6] - [03/2024]

##### Added

- Implemented Hill Climbing strategy as a simple baseline.
- Adds `use_antithetic_sampling` option to OpenAI-ES.
- Added `EvoTransformer` and `EvoTF_ES` strategy with example trained checkpoint.

##### Fixed

- Gradientless Descent best member replacement.

##### Changed

- SNES import DES weights directly and reuses code
- Made `Sep_CMA_ES` and `OpenAI-ES` use vector sigmas for EvoTransformer data collection.

### [v0.1.5] - [11/2023]

##### Added

- Adds string `fitness_trafo` option to `FitnessShaper` (e.g. `z_score`, etc.).
- Adds `sigma_meta` as kwarg to `SAMR_GA` and `GESMR_GA`.
- Adds `sigma_init` as kwarg to `LGA` and `LES`.
- Adds Noise-Reuse ES - `NoiseReuseES` - ([Li et al., 2023](https://arxiv.org/pdf/2304.12180.pdf)) as a generalization of PES. 
- Fix LES evolution path calculation and re-ran meta-training for checkpoint.

##### Fixed

- Fixed error in LGA resulting from `elite_ratio=0.0` in sampling operator logit squeeze.
- Fixed range normalization in fitness transformation - `range_norm_trafo` - Thank you @yudonglee

##### Changed

- Refactored core modules and utilities. Learned evolution utilities now in subdirectory.

### [v0.1.4] - [04/2023]

##### Added

- Adds LGA checkpoint and optimizer class from [Lange et al. (2023b)](https://arxiv.org/abs/2304.03995).
- Adds optional `init_mean` to `strategy.initialize` to warm start strategy from e.g. pre-trained checkpoint.
- Adds `n_devices` option to every strategy to control reshaping for pmap in `ParameterReshaper` (if desired) explicitly.
- Adds `mean_decay` optional kwarg to LES for regularization.

##### Fixed

- Fix missing matplotlib requirement for BBOB Visualizer.
- Fix squeezing of sampled solutions in order to enable 1D optimization.
- Fix `ESLog` to work with `ParameterReshaper` reshaping of candidate solutions.
- Fix overflow errors in CMA-ES style ES when `num_dims ** 2` is too large.

##### Changed

- Changed default gradient descent optimizer of ARS to Adam.

### [v0.1.3] - [03/2023]

- Finally solved checkpoint loading LES problem (needed `MANIFEST.in`)
- Fixed PGPE bug with regards to scaled noise.

### [v0.1.2] - [03/2023]

- Fix LES checkpoint loading from package data via `pkgutil`.

### [v0.1.1] - [03/2023]

##### Added

- Adds exponential decay of mean/weight regularization to ES that update mean (FD-ES and CMA variants). Simply provide `mean_decay` != 0.0 argument at strategy instantiation to strategy. Note that covariance estimates may be a bit off, but this circumvents constant increase of mean norm due to stochastic process nature.

- Adds experimental distributed ES, which sample directly on all devices (no longer only on host). Furthermore, we use `pmean`-like all reduce ops to construct z-scored fitness scores and gradient accumulations to update the mean estimate. So far only FD-gradient-based ES are supported. Major benefits: Scale with the number of devives and allow for larger populations/number of dimensions.
    - Supported distributed ES: 
        - `DistributedOpenES`
    - Import via: `from evosax.experimental.distributed import DistributedOpenES`

- Adds `RandomSearch` as basic baseline.

- Adds `LES` (Lange et al., 2023) and a retrained trained checkpoint.

- Adds a separate example notebook for how to use the `BBOBVisualizer`.

##### Changed

- `Sep_CMA_ES` automatic hyperparameter calculation runs into `int32` problems, when `num_dims` > 40k. We therefore clip the number to 40k for this calculation.

##### Fixed

- Fixed DES to also take flexible `fitness_kwargs`, `temperature`, `sigma_init` as inputs.
- Fixed PGPE exponential decay option to account for `sigma` update.

### [v0.1.0] - [12/2022]

##### Added

- Adds a `total_env_steps` counter to both `GymFitness` and `BraxFitness` for easier sample efficiency comparability with RL algorithms.
- Support for new strategies/genetic algorithms
    - SAMR-GA (Clune et al., 2008)
    - GESMR-GA (Kumar et al., 2022)
    - SNES (Wierstra et al., 2014)
    - DES (Lange et al., 2022)
    - Guided ES (Maheswaranathan et al., 2018)
    - ASEBO (Choromanski et al., 2019)
    - CR-FM-NES (Nomura & Ono, 2022)
    - MR15-GA (Rechenberg, 1978)
- Adds full set of BBOB low-dimensional functions (`BBOBFitness`)
- Adds 2D visualizer animating sampled points (`BBOBVisualizer`)
- Adds `Evosax2JAXWrapper` to wrap all evosax strategies
- Adds Adan optimizer (Xie et al., 2022)

##### Changed

- `ParameterReshaper` can now be directly applied from within the strategy. You simply have to provide a `pholder_params` pytree at strategy instantiation (and no `num_dims`).
- `FitnessShaper` can also be directly applied from within the strategy. This makes it easier to track the best performing member across generations and addresses issue #32. Simply provide the fitness shaping settings as args to the strategy (`maximize`, `centered_rank`, ...)
- Removes Brax fitness (use EvoJAX version instead)
- Add lrate and sigma schedule to strategy instantiation

##### Fixed

- Fixed reward masking in `GymFitness`. Using `jnp.sum(dones) >= 1` for cumulative return computation zeros out the final timestep, which is wrong. That's why there were problems with sparse reward gym environments (e.g. Mountain Car).
- Fixed PGPE sample indexing.
- Fixed weight decay. Falsely multiplied by -1 when maximization.

### [v0.0.9] - 15/06/2022

##### Added

- Base indirect encoding methods in `experimental`. Sofar support for:
    - Random projection-based decodings
    - Hypernetworks for MLP architectures
- Example notebook for infirect encodings.
- Example notebook for Brax control tasks and policy visualizations.
- Adds option to restart wrappers to `copy_mean` and only reset other parts of `EvoState`.

##### Changed

- Change problem wrappers to work with `{"params": ...}` dictionary. No longer need to define `ParameterReshaper(net_params["params"])` to work without preselecting "params". Changed tests and notebooks accordingly.
- Restructured all strategies to work with flax structured dataclass and `EvoState`/`EvoParams`.

```python
from flax import struct
@struct.dataclass
class State:
    ...
```

- The core strategy API now also works without `es_params` being supplied in call. In this case we simply use the default settings.
- Moved all gym environment to (still private but soon to be released) `gymnax`.
- Updated all notebooks accordingly.

##### Fixed

- Makes `ParameterReshaper` work also with `dm-haiku`-style parameter dictionaries. Thanks to @vuoristo.

### [v0.0.8] - [24/05/2022]

##### Fixed

- Fix gym import bug and codecov patch tolerance.

### [v0.0.7] - [24/05/2022]

##### Fixed

- Bug due to `subpops` import in `experimental`.

### [v0.0.6] - [24/05/2022]

##### Added

- Adds basic indirect encoding method in `experimental` - via special reshape wrappers: `RandomDecoder` embeddings (Gaussian, Rademacher)

##### Fixed

- Fix import of modified ant environment. Broke due to optional brax dependence.
##### Changed

- Restructure batch strategies into `experimental`
- Make ant modified more flexible with configuration dict option (`modify_dict`)

### [v0.0.5] - [22/05/2022]

##### Added

- Adds sequential problems (SeqMNIST and MNIST) to evaluation wrappers.
- Adds Acrobot task to `GymFitness` rollout wrappers.
- Adds modified Ant environment to Brax rollout.
- New strategies:
    - RmES (`RmES` following Li & Zhang, 2008).
    - Gradientless Descent (`GLD` following Golovin et al., 2020).
    - Simulated Annealing (`SimAnneal` following Rasdi Rere et al., 2015)
- Adds simultaneous batch strategy functionalities:
    - `BatchStrategy`: `vmap`/`pmap` distributed subpopulation rollout
    - `Protocol`: Communication protocol between subpopulations
    - `MetaStrategy`: Stack one ES on top of subpopulations to control hyperparameters

##### Changed

- Renamed `crossover_rate` to `cross_over_rate` in DE to make consistent with `SimpleGA`.
- Add option to add optional `env_params` to `GymFitness`, `seq_length` to addition and `permute_seq` for S-MNIST problem.
- Network classes now support different initializers for the kernels using the `kernel_init_type` string option. By default we follow flax's choice in `lecun_normal`.

##### Fixed

- Add `spring_legacy` option to Brax rollout wrappers.

### [v0.0.4] - [26/03/2022]

##### Added

- New strategies:
    - Separable CMA-ES strategy (`Sep_CMA_ES` following Ros & Hansen, 2008).
    - BIPOP-CMA-ES (`BIPOP_CMA_ES`, following Hansen, 2009)
    - IPOP-CMA-ES (`IPOP_CMA_ES`, following Auer & Hansen, 2005)
    - Full-iAMaLGaM (`Full_iAMaLGaM`, following Bosman et al., 2013)
    - MA-ES (`MA_ES`, following Bayer & Sendhoff, 2017)
    - LM-MA-ES (`LM_MA_ES`, following Loshchilov et al., 2017)
- Restart wrappers: 
    - Base restart class (`RestartWrapper`).
    - Simple reinit restart strategy (`Simple_Restarter`).
    - BIPOP strategy with interleaved small/large populations (`BIPOP_Restarter`).
    - IPOP strategy with growing population size (`IPOP_Restarter`).

##### Changed

- Both `ParamReshaper` and the rollout wrappers now support `pmap` over the population member dimension.
- Add `mean` state component to all strategies (also non-GD-based) for smoother evaluation protocols.
- Add `strategy_name` to all strategies.
- Major renaming of strategies to more parsimonious version (e.g. `PSO_ES` -> `PSO`)

##### Fixed

- Fix `BraxFitness` rollout wrapper and add train/test option.
- Fix small bug related to sigma decay in `Simple_GA`.
- Increase numerical stability constant to 1e-05 for z-scoring fitness reshaping. Everything smaller did not work robustly.
- Get rid of deprecated `index_update` and use `at[].set()`

### [v0.0.3] - 21/02/2022

- Fix Python version requirements for evojax integration. Needs to be >3.6 since JAX versions stopped supporting 3.6.

### [v0.0.2] - 17/02/2022

- First public release including 11 strategies, rollout/network wrappers, utilities.

### [v0.0.1] - 22/11/2021

##### Added
- Adds all base functionalities
