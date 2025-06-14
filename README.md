# PPO-Transformer  
笔者初学强化学习，所以很多超参数属于瞎填，见谅。  
笔者从[蘑菇书EasyRL](https://github.com/datawhalechina/easy-rl)和[PPO for Beginners](https://github.com/ericyangyu/PPO-for-Beginners)这里学习的。  
笔者实现了一个简单的PPO，整个项目就只有一个ppo.py文件  
这个文件分为三个部分 模型 环境 ppo算法  
  
首先介绍ppo算法，有三种方法计算batch_rtgs也就是rewards-to-go，笔者选择了其中性能说是比较好的。除了基本的训练逻辑以外，ppo类还负责多线程异步执行训练，可以选择任意数量的加载在cpu或者gpu上的环境进行多线程的异步学习。但异步学习容易让子模型间的策略差距越来越大，所以笔者采取了approx_kl大于一个特定值就停止训练的方法减少子环境模型间的差距。这里的approx_kl可能不会有0.01那么低，因为使用了Transformer的原因，但是训练效果还是不错的。笔者将学习率设置为一个比较低的值，因为看到网上有人说降低学习率有助于ppo去探索。  
  
接下来介绍环境，环境很简单，就是内置了一个模型，然后交互搜集各种信息，笔者在环境加了一个保存过往成功经历的功能，还有一个小设置，如果老是不成功，环境就会偏向于去探索，这是通过增加随机采样的波动实现的。  
  
最后介绍模型，笔者最开始使用一个基础的全连接的神经网络，但是学习效果不好，笔者在尝试了许多方法后依然找不到出路，最后笔者猜想是因为基础的全连接的神经网络只能看到当前的信息，无法得知过去的信息和未来的信息，评论家不一定能给出准确的价值，演员也只能按照当前的本能行动。理论上全连接的神经网络可以通过强化学习逐步寻找到出路，但是笔者观察到一个现象，模型会很快的收敛，即使这种策略不好，笔者认为这是模型对前后因果难以联系造成的，如果某个关键动作导致了全局的失败，全连接的神经网络只能通过不断的探索，最终通过rtg得知这个错误的关键动作，但可能在还没探索到这个关键动作之前，模型就只选择了一种单调的策略，而不是和上下文有关的不同策略，导致模型停止探索，陷入局部最优，笔者采用Transformer就是希望可以让模型采取的策略和当前的上下文有关。笔者没有定量的测试，但是感觉不错。  
  
关于笔者使用的Transformer有这些细节，笔者使用了双向注意力，没有使用因果掩码，也许上因果掩码会更好，因为估计这样approx_kl就能和标准的ppo一样低到0.01了，但是因果掩码不能在LinearAttention上使用，而且需要写额外的程序根据每一局的游戏记录特殊生成，笔者不想费劲了搞了，因为现在双向注意力也不错。  
目前的超参数应该还ok  

  
笔者观察到，当学习率过大，模型确实会过早收敛，收敛后approx_kl会变得很小，比如策略不稳定时会100以上，稳定时0.001以下，笔者认为也许没有必要approx_kl大于一个特定值就停止训练，所以该尝试写入了ppo_A  
笔者认为approx_kl的大小代表着模型拥有全局视野即知晓整个轨迹的情况下，在当前情况做出的动作和不知道全局视野即没有未来的视野的情况下的差异，在训练后期如果模型没有迭代的情况下approx_kl依然很小，就说明模型不会反悔，模型做出行动与未来的动作关联不大。

笔者思考这个还是用带有因果掩码的Transformer好，因为双向注意力会导致探索和训练时策略分布天然不同，并且笔者认为回放曾经的成功经验不稳定，因为过往有些成功太巧合了，而且笔者调整了多线程的更新策略，让多线程异步收集数据，到达一个阈值后小批量更新，完成整个批次更新后再迭代，笔者沿用了approx_kl大于一个特定值就停止训练，现在这个版本放ppo_B。  
  
一个直观的方式说明transformer+ppo和原始ppo的差别，如果玩飞船降落我们不提供速度信息，只提供位置，原始ppo可能无法学习，因为速度必须要和上一帧比较得到，transformer+ppo就可以通过两帧间的位置差距计算得到速度。  
笔者同时也很好奇使用上下文关联的模型在一个足够复杂的环境中训练是否会出现涌现的现象，笔者认为语言模型可以涌现是人类语言本身就是逻辑的载体，模型学会隐藏在语言背后的逻辑，如果强化学习的轨迹足够复杂，甚至包含语言信息，模型是否能学会隐藏在环境变化背后的逻辑，以此来最大化奖励。  
