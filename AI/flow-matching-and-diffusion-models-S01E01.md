# From Generation to Sampling


We represent the objects we want to generate as vectors.
Images: Height H and Width W.
3 color channels (RGB):

```javascript
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

```javascript
/**
 * Euler Method to numerically solve ODE: dx/dt = f(x, t)
 * @param {Function} vectorField - Function f(x, t) defining the ODE
 * @param {number} initialTime - Starting time (t0)
 * @param {number} finalTime - Ending time (T)
 * @param {number} initialValue - Initial condition x(t0)
 * @param {number} numSteps - Number of steps (n)
 * @returns {Array} - Array of [time, value] pairs representing the trajectory
 */
function eulerMethod(vectorField, initialTime, finalTime, initialValue, numSteps) {
    let stepSize = (finalTime - initialTime) / numSteps; // h = (T - t0) / n
    let time = initialTime;
    let value = initialValue;
    let trajectory = [[time, value]]; // Store results

    // Iteratively compute values using Euler's formula
    for (let i = 0; i < numSteps; i++) {
        value = value + stepSize * vectorField(value, time); // x_{t+h} = x_t + h * f(x_t, t)
        time += stepSize; // Update time
        trajectory.push([time, value]);
    }

    return trajectory;
}

// Example: Solve dx/dt = x with x(0) = 1 over t = [0,1] with 10 steps
function exampleVectorField(x, t) {
    return x; // Simple exponential growth dx/dt = x
}

let solution = eulerMethod(exampleVectorField, 0, 1, 1, 10);
console.log("Euler Method Solution:", solution);
```
