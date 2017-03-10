# <a name="usage"></a>Usage

The general flow for running a production lab is:

1. Specify experiments in `rl/asset/experiment_specs.json`, e.g. `"dqn", "lunar_dqn"`
2. Specify the names of the experiments to run in `config/production.json`
3. Run the lab, e.g. `grunt -prod -resume`


## Commands

We use [Grunt](http://gruntjs.com/) to run the lab - set up experiments, pause/resume lab, run analyses, sync data, notify on completion. Internally `grunt` runs the `python` command, logged to stdout as `>> Composed command: python3 main.py ...`, which is harder to use.

The useful grunt commands are:

```shell
# when developing experiments specified in default.json
grunt

# run real lab experiments specified in production.json
grunt -prod
# run lab over ssh on remote server
grunt -prod -remote
# resume lab (previously incomplete experiments)
grunt -prod -remote -resume

# plot analysis graphs only
grunt analyze -prod

# clear data/ folder and cache files
grunt clear
```

See below for the full [Grunt Command Reference](#grunt-cmd) or the [Python Command Reference](#python-cmd).


**development** mode:

- All grunt commands defaults to this mode
- specify your dev experiment in `config/default.json`
- use only when developing your new algo
- the file-sync is in mock mode (emulated log without real file copying)
- no auto-notification


**production** mode:

- append the flag `-prod` to your `grunt` command
- specify your full experiments in `config/production.json`
- use when running experiments for real
- the file-sync is real
- has auto-notification to Slack channel


## Run Remotely

If you're using a remote server, run the commands inside a `screen`. That is, log in via ssh, start a screen, run, then detach screen.

```shell
screen -S lab
# enter the screen with the name "lab"
grunt -prod -remote -resume
# use Cmd+A+D to detach from screen, then Cmd+D to disconnect ssh
# to resume screen next time
screen -r lab
# use Cmd+D to terminate screen when lab ends
```

Since a remote server is away, you should check the system status occasionally to ensure no overrunning processes (memory leaks, stuck processes, overheating). Use [`glances`](https://github.com/nicolargo/glances) (already installed in `bin/setup`) to monitor your expensive machines.

<aside class="notice">
To monitor your system (CPU, RAM, GPU), run <code>glances</code>
</aside>

<img alt="Glances to monitor your system" src="./images/glances.png" />
_Glances on remote server beast._


## Resume Lab

Experiments take a long time to complete, and if your process gets terminated, resuming the lab is trivial with a `-resume` flag: `grunt -prod -remote -resume`. This will read the `config/history.json`:

```json
{
  "dqn": "dqn-2017_02_21_182442"
}
```

The `config/history.json` is created in the last run that maps `experiment_name`s to `experiment_id`s, and resume any incomplete experiments based on that `experiment_id`. You can manually tweak the file to set the resume target of course.



## <a name="grunt-cmd"></a>Grunt Command Reference

By default the `grunt` command (no task or flag) runs the lab in `development` mode using `config/default.json`.

The basic grunt command pattern is

```shell
grunt <task> -<flag>
```

The `<task>`s are:

- _(default empty)_: run the lab
- `analyze`: generate analysis data and graphs only, without running the lab. This can be used when you wish to see the analysis results midway during a long-running experiment. Run it on a separate terminal window as `grunt analyze -prod`
- `clear`: clear the `data/` folder and cache files. **Be careful** and make sure your data is already copied to the sync location


The `<flag>`s are:

- `-prod`: production mode, use `config/production.json`
- `-resume`: resume incomplete experiments from `config/history.json`
- `-remote`: when running over SSH, supplies this to use a fake display
- `-best`: run the finalized experiments with gym rendering and live plotting; without param selection. This uses the default `param` in `experiment_specs.json` that shall be updated to the best found.
- `-quiet`: mute all python logging in grunt. This is for lab-level development only.


## <a name="python-cmd"></a>Python Command Reference

The Python command is invoked inside `Gruntfile.js` under the `composeCommand` function. Change it if you need to.

The basic python command pattern is:

```shell
python3 main.py -<flag>

# most common example, with piping of terminal log
python3 main.py -gbp -t 5 -e lunar_dqn | tee -a ./data/terminal.log;
```

The python command <flag>s are:

- `-a`: Run `analyze_experiment()` only to plot `experiment_data`. Default: `False`
- `-b`: blind mode, do not render graphics. Default: `False`
- `-d`: log debug info. Default: `False`
- `-q`: quiet mode, log warning only. Default: `False`
- `-e <experiment>`: specify which of `rl/asset/experiment_spec.json` to run. Default: `-e dev_dqn`. Can be a `experiment_name, experiment_id`.
- `-g`: plot graphs live. Default: `False`
- `-m <max_evals>`: the max number of trials for hyperopt. Default: `100`
- `-p`: run param selection. Default: `False`
- `-t <times>`: the number of sessions to run per trial. Default: `1`
- `-x <max_episodes>`: Manually specifiy max number of episodes per trial. Default: `-1` and program defaults to value in `rl/asset/problems.json`


## Lab Demo

Each experiment involves:
- a problem - an [OpenAI Gym environment](https://gym.openai.com/envs)
- a RL agent with modular components `agent, memory, optimizer, policy, preprocessor`, each of which is an experimental variable.

We specify input parameters for the experimental variable, run the experiment, record and analyze the data, conclude if the agent solves the problem with high rewards.

### Specify Experiment

The example below is fully specified in `rl/asset/classic_experiment_specs.json` under `dqn`:

```json
{
  "dqn": {
    "problem": "CartPole-v0",
    "Agent": "DQN",
    "HyperOptimizer": "GridSearch",
    "Memory": "LinearMemoryWithForgetting",
    "Optimizer": "AdamOptimizer",
    "Policy": "BoltzmannPolicy",
    "PreProcessor": "NoPreProcessor",
    "param": {
      "train_per_n_new_exp": 1,
      "lr": 0.001,
      "gamma": 0.96,
      "hidden_layers_shape": [16],
      "hidden_layers_activation": "sigmoid",
      "exploration_anneal_episodes": 20
    },
    "param_range": {
      "lr": [0.001, 0.01, 0.02, 0.05],
      "gamma": [0.95, 0.96, 0.97, 0.99],
      "hidden_layers_shape": [
        [8],
        [16],
        [32]
      ],
      "exploration_anneal_episodes": [10, 20]
    }
  }
}
```

- *experiment*: `dqn`
- *problem*: [CartPole-v0](https://gym.openai.com/envs/CartPole-v0)
- *variable agent component*: `Boltzmann` policy
- *control agent variables*:
    - `DQN` agent
    - `LinearMemoryWithForgetting`
    - `AdamOptimizer`
    - `NoPreProcessor`
- *parameter variables values*: the `"param_range"` JSON

An **experiment** will run a trial for each combination of `param` values; each **trial** will run for multiple repeated **sessions**. For `dqn`, there are `96` param combinations (trials), and `5` repeated sessions per trial. Overall, this experiment will run `96 x 5 = 480` sessions.


### Lab Workflow

The workflow to setup this experiment is as follow:

1. Add the new theorized component `Boltzmann` in `rl/policy/boltzmann.py`
2. Specify `dqn` experiment spec in `experiment_spec.json` to include this new variable,  reuse the other existing RL components, and specify the param range.
3. Add this experiment to the lab queue in `config/production.json`
4. Run `grunt -prod`
5. Analyze the graphs and data (live-synced)


### Lab Results

<div style="max-width: 100%"><img alt="The dqn experiment analytics" src="./images/dqn.png" />
<br><br>
<img alt="The dqn experiment analytics correlation" src="./images/dqn_correlation.png" /></div>

_The dqn experiment analytics generated by the Lab. This is a pairplot, where we isolate each variable, flatten the others, plot each trial as a point. The darker the color the higher ratio of the repeated sessions the trial solves._

fitness_score|mean_rewards_per_epi_stats_mean|mean_rewards_stats_mean|epi_stats_mean|solved_ratio_of_sessions|num_of_sessions|max_total_rewards_stats_mean|t_stats_mean|trial_id|variable_exploration_anneal_episodes|variable_gamma|variable_hidden_layers_shape|variable_lr
|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|:--|
1.178061|1.178061|195.726|169.2|1.0|5|200.0|196.4|dqn-2017_02_27_002407_t44|10|0.99|[32]|0.001
1.173569|1.173569|195.47|168.0|1.0|5|200.0|199.0|dqn-2017_02_27_002407_t32|10|0.97|[32]|0.001
1.152447|1.152447|195.248|170.6|1.0|5|200.0|198.2|dqn-2017_02_27_002407_t64|20|0.96|[16]|0.001
1.128509|1.128509|195.392|177.8|1.0|5|200.0|199.0|dqn-2017_02_27_002407_t92|20|0.99|[32]|0.001
1.127216|1.127216|195.584|175.0|1.0|5|200.0|199.0|dqn-2017_02_27_002407_t76|20|0.97|[16]|0.001

_Analysis data table, top 5 trials._

On completion, from the analytics, we conclude that the experiment is a success, and the best agent that solves the problem has the parameters:

- *lr*: 0.001
- *gamma*: 0.99
- *hidden_layers_shape*: [32]
- *exploration_anneal_episodes*: 10