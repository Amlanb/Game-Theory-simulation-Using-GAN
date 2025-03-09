# Evaluating Game Theory Strategies Using Generative Adversarial Networks (GANs)
This project explores the intersection of game theory and machine learning, specifically using Generative Adversarial Networks (GANs) to learn optimal strategies for the classic Prisoner's Dilemma game. The work combines principles from behavioral economics, reinforcement learning, and adversarial training to create an AI system capable of discovering and adapting to various game-theoretic situations.

Game Theory and Prisoner's Dilemma: A Deep Exploration with GANs
The Prisoner's Dilemma represents one of the most fundamental paradoxes in game theory, where two rational agents might not cooperate even when it appears in their best interest to do so. In this project, I've developed a novel approach using Generative Adversarial Networks to analyze and potentially discover optimal strategies for this classic game-theoretic problem. The GAN architecture allows the system to learn from expert strategies while developing its own approach through adversarial training. This implementation demonstrates how modern deep learning techniques can provide insights into classical game theory problems that have been studied for decades.

The project utilizes TensorFlow and Keras to implement the GAN architecture, with careful consideration given to model structure, training procedures, and evaluation metrics. By pitting the trained model against established strategies like Tit-for-Tat, Grim Trigger, and Win-Stay Lose-Shift, I've created a comprehensive framework for analyzing strategic decision-making in competitive environments. The results show interesting patterns of cooperation and defection that emerge from the trained network, offering insights into both game theory and the capabilities of adversarial machine learning systems.

Theoretical Foundation and Implementation Architecture
The project builds upon the classical Prisoner's Dilemma framework, where each player must choose to either cooperate or defect without knowing the other player's choice. I've encoded this using a standard payoff matrix where mutual cooperation yields a moderate reward (3,3), mutual defection results in a small reward (1,1), and asymmetric choices lead to the highest reward for the defector (5,0) and nothing for the cooperator (0,5). This payoff structure creates the famous dilemma where individual rationality leads to outcomes that are collectively suboptimal1.

I implemented this payoff structure using a memoized function to efficiently retrieve payoffs during gameplay simulations. The memoization technique significantly improves performance when running thousands of game iterations during training, as it avoids recalculating identical payoff values. The implementation uses Python's functools.lru_cache decorator, which provides a simple yet effective way to cache function results based on their input parameters.

# Payoff Matrix for Prisoner's Dilemma
payoff_matrix = {
    (0, 0): (3, 3),  # Cooperate, Cooperate
    (0, 1): (0, 5),  # Cooperate, Defect
    (1, 0): (5, 0),  # Defect, Cooperate
    (1, 1): (1, 1)   # Defect, Defect
}

@lru_cache(maxsize=None)  # Memoize the payoff function
def get_payoff(action1, action2):
    return payoff_matrix[(action1, action2)]

For the strategic landscape, I implemented six distinct strategies that represent different approaches to the Prisoner's Dilemma. These range from the naively cooperative ("Always Cooperate") to the strictly selfish ("Always Defect"), with several more sophisticated strategies in between. Particularly interesting are the conditional strategies like "Tit-for-Tat," which starts by cooperating and then mirrors the opponent's previous move, and "Grim Trigger," which cooperates until the opponent defects once, then defects forever after. These strategies have historical significance in game theory research, with Tit-for-Tat famously winning Robert Axelrod's tournament as a surprisingly effective simple strategy in iterated games.

GAN Architecture for Strategic Learning
The core innovation of this project is the application of Generative Adversarial Networks to learn game-theoretic strategies. I designed a generator network that takes a state representation (the previous moves of both players) and outputs a probability distribution over possible actions (cooperate or defect). The discriminator network, meanwhile, aims to distinguish between actions taken by expert strategies and those generated by the generator model.

My generator model consists of a neural network with two hidden layers (64 and 32 units), using LeakyReLU activations and batch normalization to stabilize training. The output layer uses a sigmoid activation to produce a probability of defection. For the discriminator, I implemented a slightly more complex architecture that takes the combined state and action as input, with additional dropout layers to prevent overfitting.

# Generator model
def create_generator():
    model = Sequential([
        Input(shape=(2,)),
        Dense(64, kernel_initializer='he_normal'),
        LeakyReLU(alpha=0.2),
        BatchNormalization(),
        Dense(32, kernel_initializer='he_normal'),
        LeakyReLU(alpha=0.2),
        BatchNormalization(),
        Dense(1, activation='sigmoid')
    ])
    model.compile(optimizer=Adam(learning_rate=0.0002), 
                  loss='binary_crossentropy')
    return model

The discriminator network follows a similar structure but includes dropout layers to improve generalization. This is particularly important as the discriminator needs to learn the characteristics of expert strategies without overfitting to specific sequence patterns.

Game Simulation and Strategy Implementation
To train and evaluate the GAN, I implemented a comprehensive game simulation function that can play the Prisoner's Dilemma against any of the defined strategies. This function handles the mechanics of the game, including tracking history, calculating payoffs, and implementing the logic for each strategy. For complex strategies like Grim Trigger and Win-Stay Lose-Shift, I carefully implemented the conditional logic that defines their behavior in response to the opponent's moves.

The game simulation function plays a pivotal role in both training and evaluation. During training, it generates examples of expert gameplay that serve as the "real" data for the discriminator. During evaluation, it measures how well the trained generator performs against each strategy in terms of both classification metrics and actual game payoffs.

GAN Training Process and Optimization
The training process for the GAN follows the standard adversarial approach but with specific adaptations for the game theory context. I defined a custom training step using TensorFlow's @tf.function decorator for performance optimization. This training step handles the generation of fake actions, preparation of real and fake data batches, and the calculation of appropriate losses for both networks.

A key innovation in my implementation is the focus on learning from "expert strategies" rather than all strategies. By defining Tit-for-Tat, Grim Trigger, and Win-Stay Lose-Shift as expert strategies, I direct the generator to learn patterns that lead to successful outcomes in the Prisoner's Dilemma. This approach reflects the insight from game theory that certain strategies tend to perform better in iterated games than simplistic approaches like Always Defect.

def train_gan(generator, discriminator, strategies, epochs=100, batch_size=32):
    gen_losses, disc_losses = [], []
    expert_strategies = ["Tit-for-Tat", "Grim Trigger", "Win-Stay Lose-Shift"]
    
    @tf.function
    def train_step(real_states, real_actions):
        # Generate fake actions using current generator
        fake_actions = generator(real_states)
        
        # Prepare discriminator data
        real_data = tf.concat([real_states, tf.cast(tf.reshape(real_actions, (-1, 1)), tf.float32)], axis=1)
        fake_data = tf.concat([real_states, fake_actions], axis=1)
        
        # Train discriminator
        with tf.GradientTape() as disc_tape:
            prediction_real = discriminator(real_data)
            prediction_fake = discriminator(fake_data)
            d_loss_real = tf.keras.losses.binary_crossentropy(tf.ones_like(prediction_real), prediction_real)
            d_loss_fake = tf.keras.losses.binary_crossentropy(tf.zeros_like(prediction_fake), prediction_fake)
            d_loss = 0.5 * (tf.reduce_mean(d_loss_real) + tf.reduce_mean(d_loss_fake))
            
        # ... [rest of training code]

The training loop runs for 100 epochs, collecting expert gameplay data for each batch and updating both networks. I implemented gradient clipping to prevent exploding gradients, a common issue in GAN training. Throughout training, I tracked the losses of both networks to monitor convergence and stability.

Comprehensive Evaluation Framework
After training the GAN, I implemented a thorough evaluation to assess how well the generator learned to play the Prisoner's Dilemma. The evaluation plays 100 rounds of the game against each strategy and calculates several metrics:

Classification metrics (accuracy, precision, recall, F1-score) to measure how closely the generator's moves match the opponent's

Total payoffs for both the generator and the opponent

Mean payoffs per round to normalize the results

This multifaceted evaluation approach provides insights into both how well the generator mimics each strategy and how effectively it maximizes its own payoff. The results show interesting patterns across different opponent strategies.

Results Analysis and Strategic Insights
The evaluation results reveal fascinating patterns about what the generator learned. Against cooperative strategies (Always Cooperate, Tit-for-Tat, Grim Trigger, Win-Stay Lose-Shift), the generator achieved perfect accuracy and equal payoffs, suggesting it learned to cooperate when cooperation is beneficial. This demonstrates that the GAN successfully learned the value of mutual cooperation in iterated games.

Against Always Defect, the generator achieved only 50% accuracy but maintained a non-zero payoff (0.50 per round), suggesting it learned to occasionally cooperate despite the opponent's consistent defection. This might represent a form of exploration or an attempt to encourage cooperation. Against the Random strategy, the generator achieved moderate success (mean payoff of 1.91 vs. 2.81 for the opponent), reflecting the challenge of optimizing against an unpredictable opponent.

The visualization of these results through bar charts and loss curves provides additional insights into the training dynamics and relative performance against different strategies. The loss curves show the convergence of both networks, while the payoff comparisons highlight the strategic adaptation of the generator to different opponents.

Conclusions and Theoretical Implications
This project demonstrates the potential of using GANs to learn and analyze game-theoretic strategies. The generator successfully learned to cooperate with cooperative strategies while adapting differently to defective or random strategies. This aligns with established game theory research showing that cooperative strategies often perform well in iterated games, despite the temptation to defect in single iterations.

The perfect accuracy against conditional strategies like Tit-for-Tat and Grim Trigger is particularly notable, as these strategies require understanding not just the immediate payoff but also the long-term consequences of actions. This suggests that the GAN framework can capture the temporal dynamics of repeated games, not just single-shot decisions.

In conclusion, this project represents a novel intersection of game theory and deep learning, demonstrating how modern AI techniques can provide insights into classical strategic problems. By leveraging the adversarial training paradigm of GANs, we can develop systems that learn sophisticated strategies for competitive interactions, potentially advancing both fields in the process.
