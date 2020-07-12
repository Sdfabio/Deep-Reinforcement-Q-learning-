


# Deep reinforcement Nanodegree
## Navigation Project
Fabrice Simo Defo  
July 11th, 2020

## I. Definition


### Project Overview

This project is to train an agnent to navigate and collect bananas in a large square world. A reward of +1 is provided for collecting a yellow banana, and a reward of -1 is provided for collecting a blue banana. Thus, the goal of your agent is to collect as many yellow bananas as possible while avoiding blue bananas.
The state space has 37 dimensions and contains the agent's velocity, along with ray-based perception of objects around the agent's forward direction. Given this information, the agent has to learn how to best select actions. Four discrete actions are available, corresponding to: move forward, move backward, turn left, turn right (You can check the Readme.md for a gif of an agent navigating). The task is episodic, means that the environment terminate after 300 steps



### Problem Statement

To make the agent learn, I will use a technique un Deep Reinceforcement Learning called Deep Q-Networks with some tweaks. 

First of all, deep reinforcement is a field of AI in which we use the environement and is feedback to the agent to guie the agent how to learn form it. ![Deep RL](DeepRL.PNG) [image source](https://missinglink.ai/guides/neural-network-concepts/complete-guide-deep-reinforcement-learning-concepts-process-real-world-applications/)
The procedure is simple. In our case, the agent move in the squareWorld and is in some state H (example: State H velocity 3 mph , ray vision at angle 30 degres from the west wall,...) and if he collects a banana, he receives a reward of +1. When that happens, we will change the way the agent act to emphasize more the action the agent took when he was in state H.  

In fact, when we are talking about action of an agent, the way we design it is by doing a mapping between a state A the agent might be in (Example State A velocity 2 mph , ray vision at angle 30 degres from the east wall,... ) and the action the agent SHOULD take to maximize the total reward (to collect many yellow bananas avoiding blue ones). This mappig here is called the policy \pi (\pi(state=s) = action a ). Now to approach the best policy, we use an inermediary fucntion , Q function or action state value function. And it is given by the 2nd Bellman Equation: 

![bellman Q value](Bellman2.PNG) [Richard S. Sutton and Andrew G. Barto (p.64)](http://incompleteideas.net/book/RLbook2020.pdf) 

This equation calculates the *EXPECTED* (we are in a stochastic environment) reward we will have when we are in state S and have taken action a all of that following the policy \pi (rules for action given the state of the agent when he is moving in the world). So if we can maximize this function , we will just have to take at each state s , the action a that maximizes the q-function. And this processus will be our way to approach the optimal policy \pi*.

Now with all that in mind, we have to consider a serious fact. Our state are constituted of 37 dimensions and some of them contain real numbers. Means that we have a continuum of state. So our Q function is not like a simple table with discrete state each with 4 actions. What can be a way to transform a real input of 37 real numbers (state s) into 4 real numbers representing (Q(s,left),Q(s,right),...) ? Simple neural network. So we use neural networks to approximate the Q-function. And we maximize it through refreshing the initial Q-function with new values coming from the navigation of the agent in a way it maximizes. We optimize the neural network by using Adam optimization on this error:  
![error](error.PNG)
 
 As you can see, there are 2 Neural networks one local and one target (different parameters θ_local and θ_target ). We use a way to optimize them while stabilizing the learnin

The details will be explained in the analysis part
To be brief, the agent move in the environment with random policy, collect some rewards, negative and positive. After 300 timesteps, we refresh the Q-function with these rewards to maximize it, and we restart and redo until the score is acceptable, means the agent has a good policy now. 

### Metrics and benchmark

To measure how the agent performs, we will calculate the mean score in the last 100 episodes. A score in an episode is the sum of the rewards in that episode (yellow bananas - blu ones collected). And if the mean score of the last 100 episodes attains 13, we will considered that the agent is trained enough and that the task is solved. 


## II. Analysis

### Data Exploration

We directly and only use the state given by the environment to make the agent learn. 

### Algorithms and Techniques

To approximate and maximize the Q-function, I use a deep Q-network. The process in the code is the following:

  1) First the agent do many episodes in the environment, and collect these experiences in a memory. So we have a batch of tuples (state,action,reward,next state) for many episodes. This can be seen in the "step" function in dqn_agent_project.py. Our 4 layers to approximate the Q-value in model_project.py have the following dimensions: 
  
    - fc1 (in=state_size, out=fc1_units)=(37,64) + ReLu
    
    - fc2 (in=fc1_units, out=fc2_units)=(64,64) + ReLu
    
    - fc4_advantage (in=fc2_units, out=action_size)=(64,4)
    
    - fc4_value (in=fc2_units, out=1)=(64,1)
   
   We also use discounted factor with \gamma = 0.99 to help the agent consider long term in the future results. And we select randomly our experiences (s,a,r,s') to learn from. We did this to decrease the correlation between the current policy , and the learning the agent is learning. Why ? Because we are approximating the Q-value from experiences taken from the navigation and episodes of the agent. If the network learned only from consecutive samples of experience as they occurred sequentially in the environment, the samples would be highly correlated. This will cause a learning depending on the initial random policy which dictate the sequence of states. We don't the learning to depend on the initial random policy. So taking random samples from replay memory breaks this correlation. And a bonus, we will have more efficient use of previous experience, by learning with it multiple times. In fact, multiple passes with the same data is beneficial, especially when there is low variance in immediate outcomes (reward, next state) given the same state, action pair. 
  
  2) When we have a the desired number of experiences, we optimize the parameters of the local network (θ = θ_local parameters in formula (2)) alone first by Adam with learning rate lr = 5e-4 [Adam optimization](https://arxiv.org/abs/1412.6980). And secondly, we change the target parameters ($θ^{-} = θ_target $ parameters in formula (2)) using the equation: 
  θ_target = τ x θ_local + (1 - τ) x θ_target (3)(soft-update) with τ=1e-3 . These techniques of optimization help stabilizing the learning [stabilization](https://www.nature.com/articles/nature14236.pdf). Look at this great examples in images [in this blog](https://www.freecodecamp.org/news/improvements-in-deep-q-learning-dueling-double-dqn-prioritized-experience-replay-and-fixed-58b130cc5682/)) the cowboy trying to catch the cow is like our local network which is trying to approach the target network (to reduce the error). So if we optimize both at the same time , he will difficultly catch the cow, so we we fix the target, only optimize the local network with many steps, and after optimize the target. This technique of fixing the target network is called fixed Q-targets and it can be seen in the code dqn_agent_project in "step" function with the variable UPDATE_EVERY. which means we update the targets after updatig the local network 4 times. 
  
  3) As you can see in formula 2 in the first term of the error, we don't use the action that maximizes the Q-Value for parameters θ_target , but we choose action that maximizes for parameters θ_local and evaluate Q_target with this action. At the beginning of the training we don’t have enough information about the best action to take. So when we take the maximum Q-value (which is noisy in the beginning) as the best action to take,  we can end in false positives . Then the learning become complicated because if that. So to reduce this problem of [overestimation of Q-value](https://arxiv.org/pdf/1509.06461.pdf)  , we use our DQN local network to select what is the best action to take for the next state (the action with the highest Q value) and we use our target network to calculate the target Q value of taking that action at the next state. This technique is called Double DQN
  
  4) Finally, as you can see in model_project.py , we used technique of [dueling DQN](https://arxiv.org/pdf/1511.06581.pdf) , means that we calculate the Q-value by doing the sum: 
  
 ![qvalue](qvalue.PNG)

where alpha and beta are the respective parameters for fc4_advantage which calculate the advantage function and fc4_value calculating the value function, each with his own neural network (that's why different parameters). Why ? This is because , for some states, it is unnecessary to know whether to move right  or left,... . If for example our agent  is in the middle of 4 equally espaced yellow bananas (state EXAMPLE), searching for the action that will maximize the q-value is futile in this state. The advantage function is intuitive like his name because. When we are in a state s, she helps to calculate which advantage we will have when we choose action a compared to others action a'. So if we are in a state that don't give an advantage to any actions a, then the advantage function A will be low and in the formula (4) only the Value function term will help calculating the Q value (good because we don't need more information, like in state EXAMPLE). But when there is an advantage turning left for example then A will not be as low, and this will help optimize the Q-value to turn left in this state. That's why we use 2 stream at the end of our network (fc4_advantage , fc4_value) and sum them to calculate the Q-value. To learn valuable information and not unecessary information. 






## IV. Results


### Model Evaluation and Validation
By adding each techniques one at a time and training the agent, we were able to obtain an average score of 16 around 1100 episiodes. This clearly surpass the benchmark !
![Trained Agent](results.PNG)
![resulting](code.PNG)


## V. Conclusion

### Reflection and imrpovements
This project was really intersting in terms of applying recent techniques in reinforcement techniques, ans in terms of reading papers to understand the justification of these techniques. I am always interested in the mathematical aspect to understand better. Of course when my agent was able to perform such a good score i was really happy ! Just look at this smart agent dodging blue bananas while collecting yellow ones ! 

![trainedplayer](trained.gif)



This is just the beginning of many experiences and my adventure in the reinforcement Learning field. 

Possible improvements can be: 

-[Prioritized experience replay](https://arxiv.org/abs/1511.05952) or [Double Prioritized State Recycled Experience Replay (DPSR) (July 2020)](https://arxiv.org/abs/2007.03961): These permit to select experriences from which the agent will learn the most by attributing a biggrer sample probability of experiences wha have big error (big error will cause big gradient, so big steps in learning). And the recent paper, shows us it is possible to constitute the memory of experience more precisely. Normally, we replace the experiences from new one in the memory when the agent is acting in the environement. Now the replacement can be smarter, we can discard the experiences with the lowest potential of learning , and not just the earliest because this one can have great potential of lerning (high delta error). This is prioritized replacing. Now only this can incur bias , because we will only learn from the same experiences of great delta. So to remove this bias, we do state recycling to diversify the experiences (s,a,r,s') but not by experience replacing but by state recycling with the intuition that the same state will have a good delta even with another action. This also help to explore more different next state from the original replay buffer (memory of experiences). 

-[RAINBOW](https://arxiv.org/abs/1710.02298) Which apply all these techniques in one except the DPSR

-Change the network to use CNN instead of fully connected. This will help understand the impact of the dueling DQN by analizing when we are considering advantage or not by looking inside CNN on which state the agent care about the choice of his action. (A state for example where he has to dodge blue bananas), we should be able to see a bigger illumination on the blue banana.

- Use the direct approximation of the policy instead of passing by the Q value to find the policy ([Policy based mathods](https://papers.nips.cc/paper/1713-policy-gradient-methods-for-reinforcement-learning-with-function-approximation.pdf))


