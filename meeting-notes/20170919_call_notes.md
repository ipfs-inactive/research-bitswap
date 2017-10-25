## 20170919 bitswap call notes with David Grisham and Evan Miyazono
### David's update
- talked to his friend, Everett
	- suggested psi (symbolic probabilistic programming language) - http://psisolver.org/
		- assign probability distributions to events and try to prove things from there
- David also spent a lot of time implementing things
### Evan's update
- chatted with Karthik (Caltech CMT)
	- Overview
		- First find a model
			- what is the state, what is the cost function, what are fluctuations, 
		- Consider first the extreme conditions (non-interacting or wholly interacting)
			- these are the fixed points for full defect/cooperate
			- "phase" behavior will give information about behavior between these bounds
		- define fixed points on either side in state space
			- cost function ~ action
			- classical
			- (probably microcanonical - minimize energy)
			- temperature = noise
				- dropped packets/fluctuations/network congestion
			- you can control the coupling in spin systems, but you can't 
				- JIJ (coupling) has time dependence
					- too complicated to do anything
		- then apply mean field or rg

	- MFT and RG assume ergodicity & equilibrium

	- Mean Field Theory:
		- it's a self-consistency
		- requires you can guess a form for the mean field
		- 1. assume average
		- 2. solve for new average
		- 3. show average is consistent

	- Renormalization Gruop:
		- if you have something that looks like $e^/beta H$, then you can do an RG
			- need analog for temperature
			- what is your "cost" and make that the energy
				- how do you define the ensemble
		- start with the simplest model that you can apply and then interpret the results
			- assume it's ferromagnetic - "everyone talks to everyone else the same way"
				- i.e. map your problem to the ising model and interpret analytical solutions
		- Dealing with dimensionality and range of interactions - should expect some term to disappear as you run the RG
			- infintite range is solvable
				- all pairwise interactions are constant (did on a test in 127 b?)
				- *** potentially this one - no sense of locality on the Internet
				- mean field is usually good for infinite range
			- nearest neighbor is solvable
				- 1,2 classical dimensional (2d classical = 1d quantum)
				- potentially 2d classical
			- 1/r^a hasn't been solved yet
			- David would prefer to do a power law / scale free network
				- probably would require some sort of generalized solution technique
			- other (more exotic) systems
				- spin 1 chain
					- aklt chain
				- syk model
					- all-to-all disordered majorana fermion model
				- clock models (potts)
					- other weird couplings
				- maybe check advanced books on stat mech?

	- Other mathematical/analytical/numeric methods include
		- Coupled equations for modeling
		- Percolation problem
			- have a bunch of random things happening (things moving)
			- is there a single path that lets you go through?

### Yuan Yao responded to David's email about applicaitons of Hodge decomposition to this problem
	- David will draft a response that Evan will check	
