# From Generation to Sampling


We represent the objects we want to generate as vectors.
Images: Height H and Width W.
3 color channels (RGB):

```typescript
function createImageArray(H, W) {
    let image = [];
    for (let i = 0; i < H; i++) {
        let row = [];
        for (let j = 0; j < W; j++) {
            row.push([
                Math.random(), // Red Channel
                Math.random(), // Green Channel
                Math.random()  // Blue Channel
            ]);
        }
        image.push(row);
    }
    return image;
}
```


## Data Distribution

How good an image is ~= How likely it is under the data distribution


## Generation as sampling from the data distribution

Data distribution (Pdata) is a function that takes d-dimensional real-valued input (like that imageArray) and returns a non-negative number.
This number represents the likelihood (density) of the data point occurring under the data distribution.
We can take some data point and compute it's probability density under Pdata.
If we could sample from this distribution perfectly, we would generate data points that look exactly like real ones.
However, we don't actually know this function - we just assume it exists.
We can only approximate Pdata by making a dataset from real samples.
We then need models to learn Pdata from that dataset.


## Unconditional Generation (Pdata)

One model for a fixed prompt (i.e "Dog"), so you get the same image from the same model (dogs).


## Conditional Generation (Pdata(.|y))

One model for a variable prompt y ("Dog", "Cat", "Landscape")


Generative Models generate samples from from data distribution by starting with an initial distribution.
The initial distribution is Gaussian - it's white noise:
a multivariate normal (Gaussian) distribution with the mean, or center of the distribution at zero in all dimensions,
and with a covariance matrix an identity matrix, meaning all dimensions are independent and each unit has variance.
The model transforms this random noise into meaninful samples by learning the data distribution Pdata


## Flow and Diffusion Models:

The Flow component is the basis for the Diffusion component
To grasp The Flow, first understand these concepts:
- Trajectory
- Vector Field
- Ordinary Differential Equation (ODE)

A Trajectory is the path an object follows over time.
It's a function that gives the position of a point in d-dimensional space at time t.

A Vector Field tells us how things should move at each point in space.
u(X, t) is a velocity vector that tells you the direction and speed of movement at point X.
The vector field pushes points along the trajectory.
[JS Example of a moving trajectory using a vector field](https://gist.github.com/SQLCODE917/2d6b1fd014144198d17dca697a0ca053)

An ODE is a mathematical equation that tells us how the state of the system changes over time.
i.e. `Vector Field(X[t]) = derivative(X[t])`, or
the rate of change of Xt, AKA The Trajectory, is equal to the velocity given by the vector field

Solving the ODE gives us the function X[t] that describes how the point moves.

```javascript
// Vector in d-dimensional space
type Vector = number[];

// Vector Field (ODE definition)
type VectorField = (x: Vector, t: number) => Vector;

/**
 * An Ordinary Differential Equation (ODE)
 * Defines how a system evolves over time
 */
type ODE = {
    vectorField: VectorField; // Function defining the dynamics
};

/**
 * A Trajectory is a function that returns the state at any time t
 * It is a solution to the ODE for a specific initial condition X0
 */
type Trajectory = (t: number) => Vector;

/**
 * A Flow is a collection of solutions (trajectories) for different initial conditions
 * It maps an initial condition X0 to the trajectory solution at time t
 */
type Flow = (X0: Vector, t: number) => Vector;
```

## Does a solution exist and is it unique?

The Picard-Lingelof Theorem tells us when a system that evolves over time (like an object moving or an image being transformed) has a unique and predictable path.
For this to be true, the Vector Field needs to be:
- Continuous: no suden jumps
- Smooth (Lipschitz condition): small changes in the starting position should only create small changes in the path.

We assume these conditions hold, so each starting point leads to exactly 1 trajectory.

## Numerical ODE Simulation - Euler method

The idea is to go in the direction of the vector field one small step at a time.

```typescript
// Type definitions from earlier
type Vector = number[];
type VectorField = (x: Vector, t: number) => Vector;

/**
 * Euler Method to numerically solve ODE: dx/dt = f(x, t)
 * @param {VectorField} vectorField - Function f(x, t) defining the ODE
 * @param {number} initialTime - Starting time (t0)
 * @param {number} finalTime - Ending time (T)
 * @param {Vector} initialValue - Initial condition x(t0)
 * @param {number} numSteps - Number of steps (n)
 * @returns {Array<[number, Vector]>} - Array of [time, value] pairs representing the trajectory
 */
function eulerMethod(
    vectorField: VectorField,
    initialTime: number,
    finalTime: number,
    initialValue: Vector,
    numSteps: number
): Array<[number, Vector]> {
    const stepSize = (finalTime - initialTime) / numSteps;
    let time = initialTime;
    let value: Vector = [...initialValue];
    let trajectory: Array<[number, Vector]> = [[time, [...value]]];

    // Iteratively compute values using Euler's formula
    for (let i = 0; i < numSteps; i++) {
        const derivative: Vector = vectorField(value, time);
        value = value.map((v, idx) => v + stepSize * derivative[idx]);
        time += stepSize;
        trajectory.push([time, [...value]]);
    }

    return trajectory;
}

// Example ODE: Solve dx/dt = x in 1D space
const exampleVectorField: VectorField = (x, t) => x.map(v => v); // Simple exponential growth dx/dt = x

// Example use case: Solve dx/dt = x with x(0) = [1] over t = [0,1] with 10 steps
const solution = eulerMethod(exampleVectorField, 0, 1, [1], 10);
console.log("Euler Method Solution:", solution);
```

## Where does Machine Learning come in?

### Flow Model

Takes P_init, which is a Gaussian distribution, does a transformation with an ODE, returns P_data, which is the data distribution.
Make the Vector Field that defines this ODE a Neural Network.
The architecture of the Neural Network varies by modality: whether it's images, video, proteins, etc, and will be covered later.
Vector field, which is a function of time and space, and Theta which are the parameters we can optimize over.
An ODE is deterministic, so we won't be able to generate a full distribution yet.
So what we do is make the initial distribution random: X0 is sampled from P_init.
Our goal is that X1 has the distribution P_data.

### Diffusion Model

They use Stochastic Differential Equations.
That means that their Trajectories are random.

Stochastic Process has these parts:
- Random variable X_t, random variable from 0~1
- Trajectory X, which is a function of time
- Because X is random, we can draw samples from X
- Vector Field is the same as in ODEs + the Diffusion Coefficient (sigma)
- Diffusion Coefficnent is a function of time which injects randomness (or Stochasticity) into our ODE
- Stochastic Differential Equation (SDE):
  - Initial condition X0
  - Change of Xf in time is given by a Vector Field + noise from the stochastic component (using Brownian Motion, called W)

Brownian Motion:

Is a kind of a random walk.
It's a Stochastic Process that can go for infinite time.
It always starts at 0.
It has Gaussian increments: Wt - Ws has a Normal distribuition with a variance that increases with time.
It has independent increments


Basically, the solution to an SDE is the drift term + random noise (+ a small random value from a normal distribution)
Example of an SDE adding noise to a grayscale image (represented as the 1D pixel array):

```typescript
type Vector = number[];

/**
 * Generate Gaussian noise using the Box-Muller transform.
 * @param mean Mean of the distribution (default 0)
 * @param standardDeviation Standard deviation (default 1)
 * @returns Random sample from a Gaussian distribution
 */
function generateGaussianNoise(mean = 0, standardDeviation = 1): number {
    const u1 = Math.random();
    const u2 = Math.random();
    const standardNormal = Math.sqrt(-2.0 * Math.log(u1)) * Math.cos(2.0 * Math.PI * u2);
    return mean + standardDeviation * standardNormal;
}

/**
 * Simulate forward diffusion of an image using an SDE
 * @param {Vector} pixelArray - Array of grayscale pixel intensities
 * @param {number} totalSteps - Number of steps in the diffusion process
 * @param {number} driftFactor - Drift coefficient (controls deterministic shift)
 * @param {number} noiseLevel - Noise scale factor (controls stochastic variation)
 * @returns {Vector} - Noisy image with diffused pixel values
 */
function simulateForwardDiffusion(
    pixelArray: Vector,
    totalSteps: number = 100,
    driftFactor: number = 0.0,
    noiseLevel: number = 0.05
): Vector {
    let noisyPixelArray: Vector = [...pixelArray];

    for (let step = 0; step < totalSteps; step++) {
        let timeStep = step / totalSteps;
        let driftTerm = driftFactor * timeStep; // Drift component
        let noiseStrength = noiseLevel * Math.sqrt(timeStep + 0.01); // Scale noise over time

        noisyPixelArray = noisyPixelArray.map(pixelValue => {
            let noise = generateGaussianNoise(0, noiseStrength);
            let newPixelValue = pixelValue + driftTerm + noise * 255; // Apply noise & drift
            return Math.max(0, Math.min(255, newPixelValue)); // Keep pixel values in range
        });

        console.log(`Step ${step + 1}: Added noise (σ = ${noiseStrength.toFixed(4)})`);
    }

    return noisyPixelArray;
}

// Example: Simulating forward diffusion on a grayscale image
const grayscaleImage: Vector = Array.from({ length: 100 }, () => Math.random() * 255);
const noisyImage: Vector = simulateForwardDiffusion(grayscaleImage, 50, 0.01, 0.05);

console.log("Original Pixels (first 10):", grayscaleImage.slice(0, 10));
console.log("Noisy Pixels (first 10):", noisyImage.slice(0, 10));
```

### ODE vs SDE
ODEs have Trajectories as solutions, SDEs have a random Trajectory.
To define an ODE we need a Vector Field, and for SDE we need a Vector Field and a Diffusion Coefficient.
An ODE is basically defined by a derivative of the initial condition, and for SDE, small incremental changes of the initial condition are given by small steps in the direction of a Vector Field plus small steps from the Browninan motion step which injects randomness.

### How to sample from an SDE? - the Euler-Maruyama Method!

1. We begin at an initial time of 0 with an initial state
2. To define the step size, divide the time range into n equal steps
3. Simulate over time:
    - `newState = currentState + stepSize * deterministicTrend + Math.sqrt(stepSize) * randomVariation`
    - Deterministic Trend (or drift term) represents the smooth, predictable movement
    - Random Variation (or diffusion term) adds uncertainty by injecting random noise
    - The noise is drawn from a Gaussian distribution (which is a normal distribution with a mean of 0 and variance 1)
4. Advance in time

```typescript
type Vector = number[];

type StochasticProcess = {
    computeDeterministicTrend: (state: Vector, time: number) => Vector;
    computeRandomVariation: (state: Vector, time: number) => Vector;
};

/**
 * Generates Gaussian noise for each component of a vector.
 * Uses the Box-Muller transform to sample from a normal distribution.
 * @param dimensions Number of dimensions in the vector
 * @returns A vector of random values sampled from a standard normal distribution (mean = 0, variance = 1)
 */
function generateGaussianNoise(dimensions: number): Vector {
    return Array.from({ length: dimensions }, () => {
        const randomUniform1 = Math.random();
        const randomUniform2 = Math.random();
        return Math.sqrt(-2.0 * Math.log(randomUniform1)) * Math.cos(2.0 * Math.PI * randomUniform2);
    });
}

/**
 * Implements the Euler-Maruyama method for simulating a stochastic process.
 * @param stochasticProcess The SDE model containing drift and diffusion functions
 * @param totalSteps The total number of time steps
 * @param initialTime The starting time of the simulation
 * @param initialState The initial state of the system
 * @returns An array of time-value pairs showing the evolution of the system
 */
function simulateStochasticProcess(
    stochasticProcess: StochasticProcess,
    totalSteps: number,
    initialTime: number,
    initialState: Vector
): Array<[number, Vector]> {
    const stepSize = 1 / totalSteps;
    let currentTime = initialTime;
    let currentState: Vector = [...initialState];
    let trajectory: Array<[number, Vector]> = [[currentTime, [...currentState]]];

    for (let step = 0; step < totalSteps; step++) {
        // Compute the smooth (deterministic) part of the change
        const driftTerm: Vector = stochasticProcess.computeDeterministicTrend(currentState, currentTime);

        // Compute how noise affects the system
        const diffusionCoefficient: Vector = stochasticProcess.computeRandomVariation(currentState, currentTime);
        const randomNoise: Vector = generateGaussianNoise(currentState.length).map(
            (randomSample, index) => randomSample * Math.sqrt(stepSize) * diffusionCoefficient[index]
        );

        currentState = currentState.map((value, index) => value + stepSize * driftTerm[index] + randomNoise[index]);

        currentTime += stepSize;

        trajectory.push([currentTime, [...currentState]]);
    }

    return trajectory;
}

// Example: Simulating an Ornstein-Uhlenbeck Process (Mean-reverting random motion)
const exampleStochasticProcess: StochasticProcess = {
    computeDeterministicTrend: (state, time) => state.map(value => -0.5 * value), // Pulls values toward zero
    computeRandomVariation: (state, time) => state.map(() => 0.3) // Constant noise level
};

// Initial condition (starting at 1.0)
const initialCondition: Vector = [1.0];

// Run the simulation with 100 time steps
const simulatedTrajectory = simulateStochasticProcess(exampleStochasticProcess, 100, 0, initialCondition);

// Print first 10 steps
console.log("Euler-Maruyama Simulation Output:", simulatedTrajectory.slice(0, 10));
```

### Ohrnstein-Uhlenbeck Process

Is a type of random motion (like Brownian motion), but it tends to drift back to the mean over time.
Unlike Brownian motion, there's no return to the mean, no "center" pulling you back.

## THE LAB
Lab 1: Working with SDEs - find it on [the homepage for this course](https://diffusion.csail.mit.edu/)

The exercise is to be done in Python, using pytorch, so mapping TS to Python will be covered later.
First, the review of the concepts you need:

### Part 0: Introduction

Recall: the Vector Field is the Drift Coefficient is the `uₜ(x)` in the formula.
Solving the ODE gives you the Trajectory.

The basis of both ODEs and SDEs are time-dependent Vector Fields, which are the `u`-functions that take where in space we are and where in time we are, and return the direction we should be going next.
They accept 2 parameters:
- a vector in d-dimensional space (like 2D or 3D)
- time t between 0 and 1
And they return another vector in the same-number-of-dimensions space.

```typescript
type Vector = [number, number]; // example 2D dimentsional vector

function vectorField(position: Vector, time: number): Vector {
  const [x1, x2] = position;

  // Your change function goes here, for example a rotation:
  const velocityX = -x2 * Math.sin(Math.PI * time);
  const velocityY = x1 * Math.sin(Math.PI * time);

  return [velocityX, velocityY];
}

// Usage:
const currentPosition: Vector = [1.0, 2.0];
const currentTime: number = 0.5; // halfway through time interval

const currentVelocity = vectorField(currentPosition, currentTime);
```
