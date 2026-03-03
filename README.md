# Quantitative Option Pricing & Interest Rate Modeling  
### Heston, Bates, Carr–Madan FFT, Monte Carlo & CIR Calibration  

---

## Overview

This repository contains a complete quantitative pricing and calibration framework for stochastic volatility models, jump-diffusion extensions, exotic option pricing, and interest rate modeling.

The project implements:

- Heston (1993) stochastic volatility model  
- Bates (1996) stochastic volatility with jumps  
- Lewis (2001) semi-closed Fourier pricing approach  
- Carr–Madan (1999) FFT-based pricing method  
- Monte Carlo pricing of Asian options  
- CIR (1985) short-rate model calibration  
- Yield curve construction & spline interpolation  
- Large-scale Monte Carlo rate simulation (100,000 paths)  



---

## Project Structure

1. Heston Model Calibration
2. Bates Model (Heston + Jumps)
3. Exotic Pricing – Asian Option
4. Interest Rate Modeling – CIR Framework



---

## 1. Heston Model Calibration

### Model Dynamics

Under the risk-neutral measure:

$dS_t = r S_t dt + \sqrt{v_t} S_t dW_t^S$

$dv_t = \kappa(\theta - v_t) dt + \sigma \sqrt{v_t} dW_t^v$

Calibrated parameters:

- κ — Mean reversion speed  
- θ — Long-run variance  
- σ — Volatility of volatility  
- ρ — Correlation between asset and variance  
- v₀ — Initial variance  

---

## Pricing Implementations

### Lewis (2001) Fourier Approach

- Semi-closed form integration  
- Numerical quadrature of characteristic function  
- Calibration via MSE minimization  
- Put prices obtained via put-call parity  

The results from this calibration give us the following values for the parameters in the Heston (1993) model:

$\kappa_\nu = 3.141$

$\theta_\nu = 0.129$

$\sigma_\nu = 0.00002$

$\rho = -0.0091$

$\nu_0 = 0.1$

### Carr–Madan (1999) FFT Approach

- Damped Fourier transform  
- Discrete Fourier Transform implementation  
- Efficient computation across strikes  
- Used for full volatility smile calibration  

Using both methods enables:

- Cross-validation of parameter stability  
- Numerical consistency verification  
- Performance comparison (quadrature vs FFT)  

This is to double-check the different model parameters resulting from a
calibration, repeat the same process as the previous step but using the Carr-Madan (1999) pricing approach to calibrate the Heston (1993) model.

The results from this calibration give us the following values for the parameters in the Heston (1993) model:

$\kappa_\nu =3.069$

$\theta_\nu =0.0056$

$\sigma_\nu =0.099$

$\rho =−1$

$\nu_0 =0.0138$

**Reason for different values**

Carr-Madan uses a dampening factor α to ensure the pricing function is integrable.Lewis (2001) uses an alternative formulation where the Fourier integrand is designed differently (e.g. using log prices or contours in the complex plane).

This changes the shape of the pricing function, which affects optimization — especially if the objective is sensitive to pricing errors at short maturities or deep OTM options.

---

## 2. Bates Model (Heston + Jumps)

Extends the Heston model with a Poisson jump component:

$
dS_t = ... + (J - 1) S_t dN_t$

Additional parameters:

- λ — Jump intensity  
- $\mu_J$ - Jump mean  
- $\delta_J$ — Jump volatility  

Calibrated to 60-day maturity options.

Focus areas:

- Capturing short-term skew  
- Improving fit to OTM options  
- Stability comparison vs pure Heston  

---

Now runs for 60 trading days instead of just 15. Since the longer timeframe increases the chance of sudden price jumps, I’m extending the standard Heston model by adding a Poisson jump component. This combination is known as the Bates (1996) model. My goal is to calibrate this model so that it matches the market prices of all 60-day SM vanilla options listed in the data sheet. Once the calibration is done, the desk will be able to price and manage the risk of any exotic structure maturing in about two months.

The plan to implement:
First, I’ll import the market quotes for the 60-day options — both calls and puts, covering five different strikes. For each instrument, I’ll calculate the theoretical price using the Lewis (2001) Fourier formula applied to the Bates characteristic function. To see how well the model fits the market, I’ll measure the difference between the model prices and the actual market prices using a Mean-Squared-Error (MSE) metric. Then I’ll run scipy.optimize.minimize with the L-BFGS-B method to find the set of parameters that minimises this MSE. Once the calibration is complete, I’ll plot the model prices against the market quotes and report the final parameter values.

The optimisation ran smoothly using the L-BFGS-B algorithm and converged in 113 iterations without hitting any constraints. The root mean-square error across the ten market quotes came out to about 1.22 USD, which represents roughly 6-10% of the option prices in the 11-18 USD range. When we inspected the “market vs. model” scatter plot, the fit looked consistent and monotonic, the model curve is slightly flatter than the market for deep out of the money puts and calls, but the deviations stay within about +/-1.5 USD throughout.

Looking at the diffusive volatility block, I see that k is around 3.06 per year, with theta at about 0.15, which implies a long-run volatility of about 39%.
The vol of vol parameter sigma is close to 0.39 and the correlation rho is about –0.57, both in line with what we’d expect for a single name U.S. energy stock.
The initial variance v0 is roughly 0.036, which translates to a spot volatility of about 19%, matching well with the implied level in the option strip of around 20%.
For the jump component, the estimated jump intensity lambda is around 0.65 per year, suggesting about one price jump every 18 months, the jump size has a mean of about +12% and a standard deviation of around 27%, which gives a reasonably fat tailed but realistic distribution of possible jumps.

I also tested the calibration by restarting it from seven different random seeds, and each run landed within 3% of the original objective value, a good sign that the problem is well posed and not prone to troublesome local minima.


## 3. Exotic Pricing – Asian Option

ATM Asian Call (20-day maturity) priced using:

- Risk-neutral Monte Carlo simulation  
- Full stochastic volatility path simulation  
- Pathwise averaging including S₀  
- High simulation count for variance reduction  

The final quoted price includes:

- Explicit 4% desk fee markup  

Implementation considerations:

- Vectorized simulation  
- Stable discretization scheme  
- Positive variance enforcement  

---
### Executive summary – what this project does and what the price means

The fair price is calculated for a 20-day Asian call option on SM Energy stock. An Asian option pays out based on the average stock price observed each day over the life of the contract, so it’s naturally less sensitive to sudden one-day spikes or drops than a standard European option.

This price is calculated in two main steps:
- **Calibration**, we used the Heston model and calibrated it so that it matches the current market prices of standard SM Energy options that are actually trading: this ensures the model reflects today’s market view of volatility and risk.

- **Simulation**, with those calibrated parameters, we generated a large number of possible future paths for the SM share price and its volatility. For each path, we computed the option’s payoff and averaged the results. In technical terms, this is a Monte Carlo valuation done under the risk-neutral measure.

For this pricing, it's assumed there are 250 trading days in a year. I have kept the risk-free interest rate constant at 1.50% per year, which is consistent with current USD money market levels. To make sure the simulated volatility paths stayed realistic, I used a robust full-truncation simulation method. Furthermore, to reduce statistical noise and give a reliable figure, I ran 200,000 independent scenarios, which keeps any uncertainty in the final quoted price well below a cent.

Based on the descripted work, the fair value of the Asian call option, before any fees, comes out to *\$3.73*. Statistically, the true fair price would fall within a 95\% confidence interval between *\$3.7094* and *\$3.7548* if we were to repeat this calculation over and over again.
As agreed, a 4\% service margin is applied to cover the execution costs, bringing the final price payable today to \$3.88.

## 4. Interest Rate Modeling – CIR Framework

### Term Structure Construction

- Euribor market rates  
- Yield curve bootstrapping  
- Cubic spline interpolation (weekly points)  
- 12-month horizon  

---

### CIR Model

$dr_t = \kappa(\theta - r_t)dt + \sigma \sqrt{r_t} dW_t$

Calibration performed against the interpolated term structure.

---

### Monte Carlo Simulation

- 100,000 paths  
- Daily discretization  
- One-year horizon  

Extracted metrics:

- Confidence interval for future 12M rate  
- Expected future rate  
- Pricing implications for derivatives portfolio  

---



We have Euribor rates for different maturities.


We have manually extracted these annualized percentage rates:
- 1 week   = 0.648%   -> 0.00648
- 1 month  = 0.679%   -> 0.00679
- 3 months = 1.173%   -> 0.01173
- 6 months = 1.809%   -> 0.01809
- 12 months= 2.556%   -> 0.02556

We convert them to times in years (approx).



We treat them as short deposit rates close enough to easily convert to annual yields and we want a consistent
zero-coupon curve for calibrating the CIR short-rate model.

The steps are:

1. Convert maturity and rate pairs into (T_i, R_i).
2. Interpolate them with a cubic spline, obtaining weekly points for up to 52 weeks.
3. Convert these yields to zero-coupon bond prices P(0, T).
4. Use the closed form CIR formula for P(0, T) and calibrate the model parameters
   kappa, theta, sigma, and r0 by minimizing MSE over the observed maturities.
5. Simulate the model forward with daily steps for 1 year, doing 100k paths.
   Finally, analyze the distribution for the 12-month rate at the end of the year.
