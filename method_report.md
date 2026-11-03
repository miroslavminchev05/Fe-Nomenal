# OpenADMET CYP Blind Challenge, structure track: method report

## 1. Summary

Every complex was predicted by co-folding with RoseTTAFold3 (RF3): the full-length CYP3A4
sequence, the heme as a cofactor chain (CCD `HEM`) and the ligand from its SMILES. Each ligand
was sampled many times (100 to 2,000 models). Models were repacked and minimised with Rosetta's
generic potential (`beta_genpot`) with the heme as a fixed cofactor, and one model per ligand was
chosen by an energy-and-population rule: per-atom two-state clustering of the poses relative to
the heme, the lower-energy state at every atom, and the lowest interface energy inside the
resulting set. Each submitted file was checked with the organisers' validator and PoseBusters.
The final submission is our fourth build (v4, uploaded 2026-10-25, live LDDT-PLI 0.4512).


## 2. Inputs

- Protein: the organisers' CYP3A4 sequence (UniProt P08684, 503 residues), untrimmed.
- Heme: CCD code `HEM` as its own chain. 
- Ligand: the SMILES from `cyp-challenge-TEST-BLINDED_structures.csv` as downloaded on
  2026-10-13, unchanged. 
- MSA: a pre-computed HHblits alignment (10,000 sequences), used as stored (find the MSA on
  Zenodo, DOI to follow).


## 3. Structure generation (RF3)

- RF3 (RoseTTAFold3 from github.com/RosettaCommons/foundry, container build of 2026-03-18,
  atomworks 2.1.1, default weights) run with `rf3 fold`, one input file per ligand, early stopping
  of low-confidence runs disabled.
- Run 1: 10 seeds x 10 diffusion samples = 100 models per ligand, all 20 ligands.
- Run 2: 100 seeds x 20 diffusion samples = 2,000 models per ligand for the six ligands whose
  run-1 selections were loose or came down to one model (listed in section 5).
- Each model was checked for an intact ligand graph against the SMILES; broken models were left out
  of all later steps.

## 4. Energy

Every model was repacked and minimised with Rosetta's generic potential (PyRosetta,
`beta_genpot`): side chains within 8 A of the ligand repacked, then a Cartesian minimisation of
the ligand and those side chains (backbone fixed, ligand restrained with sd 0.5 A); everything
else kept RF3's coordinates. The energies below are of the minimised models, and the minimised
models are what was submitted (section 5). Ligand parameters came from
the standard genpot route (RDKit conformer, antechamber AM1-BCC charges). The heme was built from
the CCD ideal geometry with RESP charges from the AMBER MCPB.py P450 set (Fe(III), deprotonated
propionates) and was never moved. The energy used for picking is the interface energy,
E(complex) - E(ligand moved far away), so heme-ligand contacts are included. The potential has no
Fe-N bond term, so a coordinating nitrogen is pushed about 0.1 A off the iron by the minimiser;
Fe-N distances were recorded before and after.

## 5. Pose selection

All models of a ligand are placed in one frame by superposing the heme and pocket. Then:

1. Every ligand heavy atom is a "site". At each site the models are split into two states by
   clustering that atom's position across the models (symmetry-corrected over the ligand graph,
   no ligand superposition).
2. A site is used if its two states are at least 1.5 A apart and each holds at least 5 % of the
   models (so single outlier models cannot define a state).
3. At each used site the state with the lower median interface energy wins.
4. The selection is the intersection of the winning states over all used sites. If it is empty,
   fall back to the sites with a significant energy difference (Mann-Whitney p < 0.05), then to the
   three sites with the largest effect, then to the single largest.
5. The pick is the lowest-interface-energy model in the selection whose minimised complex passes
   the organisers' structure validator and PoseBusters (`dock` configuration, receptor = protein +
   heme, any failed check counts as a fail). For six ligands (OCNT-2312966, OCNT-2315379,
   OCNT-2395342, OCNT-2395457, OCNT-2395460, OCNT-2535519) the pick was taken from the 2,000-model
   run and the candidates were further limited to the largest 1 A cluster of the selection; the
   other 14 picks come from the 100-model run.

The submitted file is the minimised complex (chain A protein, chain B heme, chain C ligand `LIG`).
For two ligands the minimised ligand failed the organisers' connectivity check (a squeezed
cyclobutane in OCNT-2395789 and a squeezed cyclopropane in OCNT-2535519, where a distance-based
bond guess adds a bond), so the raw RF3 coordinates were written instead.

## 6. Other things tried that did not beat v4

- AlphaFold 3, Protenix 2.0, Boltz-2, Chai-1 and ESMFold2 ensembles, alone and pooled with RF3 in
  a method-blind version of the same picking rule.
- Refitting the genpot term weights on five recent CYP3A4 cryo-EM complexes (Orta et al. 2026,
  PDB 38FU to 38FY).
- Extending the 2,000-model run and the same pick rule to five more ligands (OCNT-2313397,
  OCNT-2395328, OCNT-2395415, OCNT-2395464, OCNT-2395789): the upload with those five swapped
  scored LDDT-PLI 0.4445 against v4's 0.4512 (leaderboard, 2026-10-25), so their 100-model picks
  were kept.
- Picking the most central model of the largest pose family instead of the lowest-energy model
  (lost 0.70 LDDT-PLI summed over four ligands when tested).

## 7. Limitations

- Three ligands carry the wrong stereochemistry relative to the SMILES the organisers published on
  2026-10-24 (two inverted centres, one ring cis/trans). Corrected files scored slightly lower on
  the leaderboard, so the originals were kept. The scoring is graph-based and does not check
  stereo, but the positions are affected.
- The energy function has no explicit Fe coordination term, so on-Fe and off-Fe poses are not
  compared fairly. Reassuringly, RF3 itself places a type II head (a nitrogen coordinating the
  iron) for about two thirds of the ligands; the exception is one ligand whose nitrogens both
  carry a hydrogen and should not coordinate.
- For the ligands that do not coordinate the iron, RF3 spreads its models over many pose families
  with no energy separation between them; for those the pick is close to a coin flip.
