---
title: Beluga Airbus Competition
subtitle: Web Service Based Evaluation
---

# Content of the Repository

This repository hosts a module of the competition’s evaluation system. The infrastructure is built on dockerized web services, enabling the execution of competitors’ algorithms and the collection of results. The module is configured for local execution, allowing participants to test their code before preparing it for submission.

Specifically, the repository contains:

* A template for a competitor submission (trivial solution code wrapped in a dockerized web service)
* The web service that controls the evaluation process (also in a Docker container)
* A Docker Compose configuration file to simplify the cofiguration and execution of the evaluation system

# Requirements and Dependency Configuration

To run this system, you must install [Docker](https://www.docker.com). All additional dependencies are managed within the containers. Familiarity with Docker is recommended before setting up the evaluation system, as it will better prepare you to make any necessary modifications.

If you wish to install additional dependencies (or if you are encountering issues), you may need to make adjustments to the container configuration files.
In particular, the image buiding process for the _evaluator container_ is defined in `evaluator/Dockerfile`. During the image construction, this will:

* Install basic dependencies for the web service wrapper (these are in `evaluator/requirements.txt`)
* Install the default dependencies for the competition toolbox (in `evaluator/src/tools/requirements.txt`)

There should be no need to chance these files, as they have no direct impact on the submission code.

The configuration for the _submission container_ can be found in `submission/Dockerfile`. During the image construction, this will:

* Install basic dependencies for the web service wrapper (these are in `submission/requirements.txt`)
* Install the default dependencies for the competition toolbox (in `submission/src/tools/requirements.txt`)

Additional dependencies for the competitor's own submission can be installed by either:

* Making changes to either of the two `requirements.txt` file in the submission configuration
* Making changes directly to the submission Dockerfile

# Preparing a Submission for Evaluation

The submission template consists of a basic wrapper for subclasses of the main competition API. The provided implementation simply wraps naïve randomized solvers, which allows to make a "dry run" of the testing system before starting to integrate the competitor's code. If fact, doing such a dry run is highly recommended.

In particular, competitors should make sure that their code subclasses either the `DeterministicPlannerAPI` or `ProbabilisticPlannerAPI` from the `evaluator.planner_api` module in the toolkit.

Then, they should edit the `submission/src/business_logic.py` file in two parts, both marked with a comment including the string "SECTION TO BE EDITED BY THE COMPETITORS".

* The first section is at the beginning of the file and is meant to contain all the `import` statements needed to run the competitor's code
* The second section is in the constructor of the `CompetitorModelBusinessLogic` class and is meant to build the competitor's implementation of either the `DeterministicPlannerAPI` or `ProbabilisticPlannerAPI`

These two small edits should be sufficient: the WS wrapper provided in the template will call the planner API and thus execute the competitor's code.

# The Evaluator

The evaluator WS is a replica of the one used for the actual evaluation infrastructure in the competition. There should be no need to modify the code, but several parameters of the evaluation process can be configured for convenience.

This can be done by editing the file `evaluator/conf.toml`, which includes the following fields:

* In the `[infrastructure]` section, there are two configuration options:
  - `send_to_orchestrator`: this should alway be set at `false`, since it is only needed for the online evaluation system
  - `resume`: if `1`, starting the evaluation for a submission id preserves any past results and the submission WS is called only on the problems that had not yet been solved. If `0` (the default behavior), all past results for a submission id are deleted before evaluating the same submission again. There's a third allowed value, i.e. `2`, which is for internal use.
  - `reboot_time_limit`: leave this to the default (it is used in the online evaluation system to avoid repeating a full evaluation when the submission container is killed -- e.g. due to exceeded resource limits)
* The `[evaluation]` section includes multiple options:
  - `problem_dir`: this specifies the directory with the benchmark for the deterministic problem. It is recommended to keep it to its current value or to keep it commented; both cases result in the training dataset being used
  - `problem_dir_prob`: this specifies the directory with the benchmark for the probabilistic problem. It is recommended to keep it to its current value or to keep it commented; both cases result in the training dataset being used
  - `time_limit`: the time limit, in seconds, for the construction of each plan (deterministic challenge) or for generating actions in _all_ the simulations for a single problem (probabilistic challenge).
  - `max_steps`: the maximum number of steps (per jig) to be executed in any plan or simulation; affects both the deterministic and the probabilistic challege
  - `nsamples`: the number of samples (simulations) for the probabilistic challenge
  - `alpha`: the value of the _alpha_ parameter in the scoring function
  - `beta`: the value of the _beta_ parameter in the scoring function
  - `seed`: the seed for the main random number generator

By default, all parameters are configured for quick dry runs. In particular, the step limit and number of samples are kep intentionally very small. Once your dry run is successful, you can perform a more realistic test by just commenting all options in the `[evaluation]` section, except for the `seed` parameter: commented parameters are set to their default values, which are the ones used in the competition.

The exception is the `seed` parameter that, if left commented, enables the default numpy RNG initialization. During the official evaluation a fixed (undisclosed) seed will however be used. During local runs, we recommended using a fixed seed for predictable results or easier debugging, and commenting the seed if the goal is testing the robustness of the submission code.

# Starting the Evaluation

Both the evalutor and submission WS images can be built as usual in Docker Compose, i.e. with the command:

```sh
docker compose build
```

After the images are built, both the evaluator and submission WS should be started with the command:

```sh
docker compose up
```

The actual evaluation does not start until a request is sent to the submission WS. For the deterministic challenge, this can be done by using curl as follows:

```sh
curl http://0.0.0.0:8088/run\?submission_id\=-1\&competition_type\=scalability_deterministic\&optit_endpoint\=http://0.0.0.0:81\&competitor_model_endpoint\=http://submission:80 -X POST
```

For the probabilistic challenge, use instead:

```sh
curl http://0.0.0.0:8088/run\?submission_id\=-2\&competition_type\=scalability_probabilistic\&optit_endpoint\=http://0.0.0.0:81\&competitor_model_endpoint\=http://submission:80 -X POST
```

The `submission_id` parameter can be set to any integer number for a local evaluation. In the example command, we use -1 for a deterministic evaluation and -2 for probabilistic one, so that running one command does not overwrite the output of the other. Be mindful, however, that if you set the configuration parameter `resume` to `true`, only the past results for the same submission id will be recovered.

The web service containers can be removed as usual in Docker Compose by running:

```sh
docker compose down
```

# Inspecting the Results

During the evaluation, the messages exchanged between the two WS, as well as the final outcomes, will be stored as json files in the `evaluator/res` and `submission/res` directories.

The outcome of the evaluation in particular will be in `evaluator/res/output`. This will contain:

* A `score.json` file, with the aggregated score for the tested submission. This is the file that will be use by the official evaluation system to rank the submission.
* Outcomes for the individual problems/simulations in a directory having the submission id as a name (if the example commands are used, this is -1 for the deterministic challenge and -2 for the probabilistic challenge)

Remember that the competition uses separate sets of problems: a "training" set provided to the competitors (and included in this repository), a "validation" set to send an outcome for every submision, and a "test" set used to compute the final ranking after the submission proces is closed.
