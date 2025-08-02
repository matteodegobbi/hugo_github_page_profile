+++
date = '2025-08-02T18:19:21+02:00'
draft = false 
title = 'Dynamic Producer Consumer 📈📉'
categories=["Low level programming"]
image="/example_plots/fixed_delay/pid_1000_1000/prod_delay.png"
tags=["c","pthreads","concurrent programming"]
+++

# Producer-Consumer problem with dynamic rate adjustment

In this repo I implemented the producer consumer problem with dynamic rate 
adjustment carried out with an actor.

The consumer is a thread that consumes messages at a given fixed rate, that is, with a given delay from a shared buffer, simulating the consumed message usage.
The producer is a thread that creates messages and writes them to a shared buffer, the rate of the producer is controlled by another thread called the actor.

An actor separate from producer and consumer periodically checks the message queue length and if the length is below a given threshold, it will increase the production rate. Otherwise (i.e. the message length is above the given threshold), it will decrease the production rate.
In this case the message queue is implemented with a circular queue (or ring buffer). 

---

The main idea is to try to keep the buffer always half-full.
Meaning the objective is to have buffer utilization always close to 50%, this leaves room in both ways:
if the consumer is delayed we will still  have room for inserting new data, and if the producer is delayed we still have data to consume for a while.
This way it maximizes the production rate until the buffer utilization goes over 50%. Any other threshold could be chosen and the program still works (e.g. I tried also with 70%).

I tried two approaches, both inspired by control systems, the control is carried out by changing a delay in the producer thread, this will dynamically adjust the rate of the producer to try to keep the buffer utilization at 50%.

The two controllers are:
1. A simple threshold based control system
2. A PID controller

Both systems share the code for the producer and the consumer.
The actor that carries out the control is different in the two solutions.
In both cases the actor runs at a fixed rate (e.g. every 1ms or 0.5ms etc.) and it checks the current buffer utilization, it then uses this information to adjust the producer's rate.
More information about the two methods will be given below.

It's important to have the actor's rate not too high or too low.
If the delay is too high then the actor does not have time to react to changes
in buffer utilization, leading to oscillations between the buffer being
too full and too empty.
Inversely if the delay is too small the actor will enter the critical
section too often, making the producer and consumer wait 
to enter their critical section and degrading the perfomance of the two.

Unfortunately we cannot fix a rate for the actor for every task and every 
system and this will need to be tuned depending on the system used.

Another important detail is the length of the buffer:
if the buffer is too small and the producer (or consumer) is carrying out
some operation, the other thread will empty (or fill) the buffer and 
we don't have enough room to provide some leeway for the two threads 
to do their work while the other one is waiting or performing some 
other operations, hampering the concurrency of the system.
In most of my experiments I tried buffer sizes of 100000 and 1000000.
This problem can be witnessed with a buffer size of just 1000 or 100 
the buffer utilization oscillates between 0% and 100%.
The buffer length also influences at what rate we can run the actor,
a bigger buffer allows us to run the actor less often, meaning the actor 
adds less overhead to the system.

---

## Control methods
In this section I explain the two methods I implemented to try to maximize the production rate while keeping the buffer utilization under 50%.

### Naive threshold based method

The first method I implemented is called `naive_actor` in the code and it loops over the following code:
```c++
    pthread_mutex_lock(&mutex);
    int count = (writeIdx - readIdx + buffer_size) % buffer_size;
    float percent = (float)count / (float)buffer_size;
    if (percent < 0.45) {
      producer_delay *= 0.8;
      producer_delay += 1; // to avoid having delay==0
    } else if (percent > 0.55) {
      producer_delay *= 1.2;
    }
    producer_delay =
        producer_delay > 1e6 ? 1e6 : producer_delay; // clip if above 1s
    int last_delay = (int)producer_delay;
    pthread_mutex_unlock(&mutex);
```
this is a very simple implementation that just check whether
the percent utilization of the buffer has gone above 55% or
below 45% and decreases or increases the producer's rate by 20%.
This approach is similar to bang-bang control in control theory 
since it only activates when it's between two thresholds,
in this case the output is like a binary on/off that either 
increases or decreases the delay by 20% independently of how far the 
buffer utilization is from 50%.

The producer delay is clipped to be 1 second at most, this is not necessary with the normal setup with fixed consumer
delay but later I will explain a variation I tried where the consumer's delay is generated by a Cauchy random variable
and in this case this naive controller can be unstable and create high delays above 1 second in the producer.

When using the PID controller that I will explain later, this step is not needed, because even when the consumer's delay is 
generated by a Cauchy r.v. the system is still stable without clipping.


In the figure below we can see how the naive controller performs, 
the buffer utilization reaches 50%, then it overshoots
to ~57% and then starts oscillating between ~43% and ~57%,
this is expected for two reasons:
1. The controller only works if the utilization goes outside the range [0.45,0.55],
2. When it changes the delay it only reacts by changing it by a fixed percentage
of the previous value. It does not use the distance from the target utilization
or the rate of change of the error, 
meaning if the current utilization is 56% it will react in the same way if
the previous utilization was 55% or if it was 90%. But it should react differently,
if the previous utilization was 56% and it's still 56% we probably need to increase the delay,
but if it was 90% it means the consumer has increased its speed for whatever reason so
we probably won't need to increase the delay.

This 2nd point made me think if I can incorporate in some way the rate of change,
and at this point I realized that I can use a PID controller, this way I can use the
rate of change through the derivative and I can also tune the parameters more easily,
since I just need to change 3 numbers.
Also using a PID controller the proportional term will be proportional to the error
reacting differently if the error is small or large, the naive controller always reacts
at the same speed independently from the magnitude of the error.

<figure>
  <img src="/example_plots/fixed_delay/naive/percent_utilization.png" alt="Image 1">
  <img src="/example_plots/fixed_delay/naive/prod_delay.png" alt="Image 2">
  <figcaption><strong>Figure 1:</strong> Buffer utilization percentage and producer's delay when using the naive controller</figcaption>
</figure>

### PID controller

To implement the PID controller I changed the actor code and made the function `pid_actor` which changes the main loop to this:
```c++
float prev_error = 0.0;
float integral = 0.0;

while (keep_running) {
    pthread_mutex_lock(&mutex);
    int count = (writeIdx - readIdx + buffer_size) % buffer_size;
    float percent = (float)count / (float)buffer_size;

    float error = 0.50 - percent;

    integral += error;
    float derivative = (error - prev_error);
    float output = k_p * error + k_d * derivative + k_i * integral;
    prev_error = error;

    producer_delay = producer_delay - (int)(output);
    producer_delay = producer_delay < 0 ? 0 : producer_delay; // clip if below 0

    int last_delay = producer_delay;
    pthread_mutex_unlock(&mutex);

    // write buffer utilization and producer delay rate
    fprintf(fp, "%.2f,%d\n", percent, last_delay);
    usleep(ACTOR_DELAY);
}
```
The actor computes the error as the difference between the 0.50 (the wanted buffer utilization) and the current buffer utilization.
Then this error is used to compute the output of the controller that will be subtracted from the current delay in a feedback loop.
We have that the output is made up of three terms:
1. Proportional term which is proportional to the error itself, this is similar to the naive controller
2. Integrative term, which means proportional to the sum of past errors
3. Derivative term, meaning proportional to the difference between the current and previous error. (This is the part that incorporates the rate of change)

The multiplicative constants $K_p,K_i,K_d$ are read from a configuration file.

In this case we don't need to clip the delay to be below one second as the system is already stable, but we do need to clip it to not go below 0 as a negative delay makes no sense.

This PID actor works much better without oscillations and it just requires to try a few combinations of the parameters to work.
Some values of $K_p,K_i,K_d$ I found that work well are $1000,0,1000$, $1000,0,10000$ or $10000,0,10000$.

The constant $K_d$ of the derivative term has the effect of dampening oscillations and overshoots, but if it's increased too much 
it can make the system too slow to react to changes of regimes and to initially reach the 50% utilization.

I found that the integral term is usually not needed as there isn't a drift in time from the wanted output. The only set
of parameters I found an instance in which the integral term helps:
with $K_p=100,K_d=10000$ the system has a small steady state error
so adding a small integral term of $K_i=0.01$ removes this steady state error at the cost of some oscillation. Since this
steady state error can also be removed by increasing $K_p$ without introducing oscillations I never wound up using the 
integral term but I left it implemented because in some systems it could be useful to counteract systemic noise.

This example of adding the effect of adding $K_i=0.01$ can be seen in the figure below.

<figure>
  <img src="/example_plots/integral_steady_state/1percent_utilization.png" alt="Image 1">
  <figcaption><strong>Figure 2a:</strong> Buffer utilization percentage with PID controller and Kp=100,Ki=0,Kd=10000. We can notice that
the steady state of utilization is slightly above 50%, around 51% and no oscillation occurs.
</figcaption>
  <img src="/example_plots/integral_steady_state/2percent_utilization.png" alt="Image 1">
  <figcaption><strong>Figure 2b:</strong> Buffer utilization percentage with PID controller and Kp=100,Ki=0.01,Kd=10000. We can notice that
the utilization oscillates around 50%.
</figcaption>
  <img src="/example_plots/integral_steady_state/3percent_utilization.png" alt="Image 1">
  <figcaption><strong>Figure 2c:</strong> Buffer utilization percentage with PID controller and Kp=1000,Ki=0,Kd=10000, the system quickly reaches 50% 
utilization and does not oscillate.
</figcaption>
</figure>

---

Below I will show some plots when using the PID controller with different constants.

Firsty let's see with just a proportional controller:
<figure>
  <img src="/example_plots/fixed_delay/pid_100_0/percent_utilization.png" alt="Image 1">
  <img src="/example_plots/fixed_delay/pid_100_0/prod_delay.png" alt="Image 2">
  <figcaption><strong>Figure 3:</strong> Buffer utilization percentage and producer's delay when using the PID controller
with Kp=100,Ki=00,Kd=0, we have oscillations in both the utilization and the producer delay </figcaption>
</figure>

Since with just a proportional controller like in the previous figure there are oscillations
we can try to add a derivative term to try to dampen the oscillations:
<figure>
  <img src="/example_plots/fixed_delay/pid_100_1000/percent_utilization.png" alt="Image 1">
  <img src="/example_plots/fixed_delay/pid_100_1000/prod_delay.png" alt="Image 2">
  <figcaption><strong>Figure 4:</strong> Buffer utilization percentage and producer's delay when using the PID controller with Kp=100,Ki=0,Kd=1000,
the oscillations decrease in magnitude over time. Eventually the producer's delay matches the consumer's.
</figcaption>
</figure>

We can now try to increase Kp and/or Kd to try to see if we can get better outcomes in terms of reaching
steady state faster and reducing overshoot and oscillations:

<figure>
  <img src="/example_plots/fixed_delay/pid_1000_1000/percent_utilization.png" alt="Image 1">
  <img src="/example_plots/fixed_delay/pid_1000_1000/prod_delay.png" alt="Image 2">
  <figcaption><strong>Figure 5:</strong> Buffer utilization percentage and producer's delay when using the PID controller with Kp=1000,Ki=0,Kd=1000,
the overshoot is decreased and the systems reaches steady state quickly, still there are oscillations in the producer's delay.
</figcaption>
</figure>

To try to reduce the oscillations in the producer's delay I finally increased both Kp and Kd to make the system settle fast
but also reduce the oscillations in the controlling signal.


<figure>
  <img src="/example_plots/fixed_delay/pid_10000_10000/percent_utilization.png" alt="Image 1">
  <img src="/example_plots/fixed_delay/pid_10000_10000/prod_delay.png" alt="Image 2">
  <figcaption><strong>Figure 6:</strong> Buffer utilization percentage and producer's delay when using the PID controller with Kp=10000,Ki=0,Kd=10000,
the utilization doesn't overshoot and reaches steady state quickly, the oscillations in the producer's delay happen only at the beginning and quickly go away.
</figcaption>
</figure>

---

## Variable rate consumer using Cauchy distributed random delay

Finally I decided to see if the two systems can deal with a consumer that has a very variable
rate, to do this is decided to generate the delay of the consumer with a Cauchy random variable with center 100 and scale 
parameter 100.

I choose a Cauchy rv because it has very high variability and is unpredictable, the median of this delay will still be 100 microseconds but 
much higher delays can be generated, still I clipped the delay to not be more than 1e6 microseconds (1 second) to avoid very long wait times.

Just to see how variable this delay is we can run this R script:
```python
x_gaussian = rnorm(mean=100,sd=100,n=1000)
x_cauchy = rcauchy(100,100,1000)
print(median(x_gaussian))
print(median(x_cauchy))
print(max(x_gaussian))
print(max(x_cauchy))
```
with a run of  this script we get that the medians are 107.7077 for the gaussian and 106.806 for the Cauchy.
But the maxes are very different: 475.0348 for the gaussian and 55947.0 for the Cauchy. And this is with only 1000 samples,
increasing the number of samples would highlight the difference even further as Cauchy r.v.'s don't have a finite variance.

So the system needs to be able to act quickly in order to counteract this kind of spikes in delays.

Now I will show some plots overviewing the behaviour of the two controllers when dealing with this Cauchy delay.

<figure>
  <img src="/example_plots/cauchy/naive_cauchy/percent_utilization.png" alt="Image 1">
  <img src="/example_plots/cauchy/naive_cauchy/prod_delay.png" alt="Image 2">
  <figcaption><strong>Figure 7:</strong> Buffer utilization percentage and producer's delay when using the naive controller, the consumer's delay
is generated by a Cauchy r.v.
</figcaption>
</figure>


<figure>
  <img src="/example_plots/cauchy/pid_cauchy_10000_10000/percent_utilization.png" alt="Image 1">
  <img src="/example_plots/cauchy/pid_cauchy_10000_10000/prod_delay.png" alt="Image 2">
  <figcaption><strong>Figure 8:</strong> Buffer utilization percentage and producer's delay when using the PID controller with Kp=10000,Ki=0,Kd=10000,
the consumer's delay is generated by a Cauchy r.v.
</figcaption>
</figure>

As we can see from the previous Figures 7 and 8 the two controllers exhibit very different behaviours.
The naive controller goes from delay 0 to 10 seconds of delay (this is the reason why after I added the clipping to 1 second)
in some systems (e.g. my laptop) this is very drastic and can hang the system for a long time, the buffer utilization is always between
40% and 60% but it's more unstable.

With the same setup the PID controller is much more stable having spikes only when the consumer has a long delay
and quickly reaching 50% again by adjusting the producer rate. The controller also doesn't reach very long delays as it can 
respond in a more stable manner, reaching at most 60000 microseconds delay against 1e7 microseconds (or more) of the naive controller.

---

## How to run the experiments 

You need to have installed `gcc` and `make` in a POSIX compatible machine with pthreads (I tested the code on Ubuntu 24.04 LTS on a x86 machine and MacOS Sequoia 15.5 on a ARM machine)

For the plotting you need to have installed `python` with the libraries `pandas` and `matplotlib`.

To run the experiment you can set the PID constants in the text config file `pid_constants.txt` and run
```bash
# to run with fixed delay
make run-and-plot MODE=1 
# to run with Cauchy delay
make cauchy-run-and-plot MODE=1
```
The `MODE` argument when set to 1 (default value) runs with the PID controller and when set to 0 runs with the naive controller.

To set the durations of the experiment or the buffer size you can change the Makefile.










