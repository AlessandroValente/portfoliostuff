# odin plots

source: Big Beautiful Model, ODIN-Model 1.1.14. orbit: 600 km orbit mean, two sun-synchronous planes, LTAN 06:00 and 18:00.

## mission (MISSION.m, run 6 Nov 2026)
shadowMission.png | longest shadow of each day over five years, both planes; peaks 21.16 min | use
shadowAssumptions.png | same data, larger type for slides | skip, duplicate
arrayOrbit.png | array power at the bus over one orbit: first week, worst orbit on the Sun, worst orbit 63.3 deg off | use
batteryMission.png | lowest charge in the worst shadow and capacity left over five years, against the 40% floor | use
windowsCharge.png | charge through 60 min observation windows at 63.3 deg, worst week | maybe
densityLocalTime.png | NRLMSISE-00 density against local solar time at 600 km, three solar forecasts | maybe
decayB.png | altitude after five years against ballistic coefficient, three forecasts, 550 km line | use

## uncertainty (UNCERTAINTY.m, run 9 Nov 2026, 100000 draws, 14 of 14 checks passed)
altitudeBand.png | orbit-mean altitude over five years, 3 sigma bands, three forecasts | use
altitudeBandTop.png | same, wide format | skip, duplicate
ltanBand.png | LTAN change over five years, 3 sigma band | maybe
shadowRange.png | range of the longest shadow by forecast and RAAN window, against the 35.8 min sizing shadow | use
alongTrack.png | 3 sigma along-track error of a prediction, GNSS fix against drag | maybe
contactAltitude.png | ground station time in view a day against altitude, 190 min line | maybe
convergence.png | percentile change against number of draws | skip, check
decayCheck.png | decay table against dragDecay | skip, check
pathCheck.png | ode45 against Clohessy-Wiltshire | skip, check

## orbit-trade (DR1, 2 Oct 2026): old six-orbit trade, team did not move forward
fig_dr1_shadow_bars.png, fig_dr1_ltan.png | skip unless the trade stays on the site

## validation (Sep 2026): model checks, still valid
eclipseCrossCheck.png | root-solved shadow times against Aerospace Toolbox | maybe
numericalCrossCheck_altitude_365d.png | J2 model against high-fidelity propagator, a year | maybe
groundContacts.png | ground station passes | maybe
lighting_*.png, sixOrbits.png | old trade cases | skip
