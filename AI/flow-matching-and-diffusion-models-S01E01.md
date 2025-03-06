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

An ODE is a mathematical equation that tells us how a system changes over time.
The rate of change of Xt (the trajectory) is equal to the velocity given by the vector field ut.
Given an initial condition X0, solving the ODE traces the entire trajectory.
