# WIFI
Simulations  .11ax, .11mc, .11be et al  - Exploration

"""  Work still under Progress """"


from ns3gym import ns3env
import numpy as np
import tqdm
import subprocess
from comet_ml import Experiment
import matplotlib.pyplot as plt
from collections import deque
import time
from comet_ml import Experiment, Optimizer



class Teacher:
    """Class that handles training of RL model in ns-3 simulator
    Attributes:
        agent: agent which will be trained
        env (ns3-gym env): environment used for learning. NS3 program must be run before creating teacher
        num_agents (int): number of agents present at once
    """

    def __init__(self, env, num_agents, preprocessor):
        self.preprocess = preprocessor.preprocess
        self.env = env
        self.num_agents = num_agents
        self.CW = 16
        self.action = None              # For debug purposes

    def dry_run(self, agent, steps_per_ep):
        obs = self.env.reset()
        obs = self.preprocess(np.reshape(obs, (-1, len(self.env.envs), 1)))

        with tqdm.trange(steps_per_ep) as t:
            for step in t:
                self.actions = agent.act()
                next_obs, reward, done, info = self.env.step(self.actions)

                obs = self.preprocess(np.reshape(next_obs, (-1, len(self.env.envs), 1)))

                if(any(done)):
                    break

    def eval(self, agent, simTime, stepTime, history_length, tags=None, parameters=None, experiment=None):
        agent.load()
        steps_per_ep = int(simTime/stepTime + history_length)

        logger = Logger(True, tags, parameters, experiment=experiment)
        try:
            logger.begin_logging(1, steps_per_ep, agent.noise.sigma, agent.noise.theta, stepTime)
        except  AttributeError:
            logger.begin_logging(1, steps_per_ep, None, None, stepTime)
        add_noise = False

        obs_dim = 1
        time_offset = history_length//obs_dim*stepTime

        try:
            self.env.run()
        except AlreadyRunningException as e:
            pass

        cumulative_reward = 0
        reward = 0
        sent_mb = 0

        obs = self.env.reset()
        obs = self.preprocess(np.reshape(obs, (-1, len(self.env.envs), obs_dim)))

        with tqdm.trange(steps_per_ep) as t:
            for step in t:
                self.debug = obs
                self.actions = agent.act(np.array(logger.stations, dtype=np.float32), add_noise)
                # self.actions = agent.act(np.array(obs, dtype=np.float32), add_noise)
                next_obs, reward, done, info = self.env.step(self.actions)

                next_obs = self.preprocess(np.reshape(next_obs, (-1, len(self.env.envs), obs_dim)))

                cumulative_reward += np.mean(reward)

                if step>(history_length/obs_dim):
                    logger.log_round(obs, reward, cumulative_reward, info, agent.get_loss(), np.mean(obs, axis=0)[0], step)
                t.set_postfix(mb_sent=f"{logger.sent_mb:.2f} Mb", curr_speed=f"{logger.current_speed:.2f} Mbps")

                obs = next_obs

                if(any(done)):
                    break

        self.env.close()
        self.env = EnvWrapper(self.env.no_threads, **self.env.params)

        print(f"Sent {logger.sent_mb:.2f} Mb/s.\tMean speed: {logger.sent_mb/(simTime):.2f} Mb/s\tEval finished\n")

        logger.log_episode(cumulative_reward, logger.sent_mb/(simTime), 0)

        logger.end()
        return logger

