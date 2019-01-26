vpath %.rds .:..
vpath %.R .:..

-include local.mk

wrap = $(addsuffix $(3),$(addprefix $(1),$(2)))

default: effstat

projref.rda: projref.R
	Rscript $^ $@

# INJESTING DB

# create data.table translations of baseline / intervention results
# includes the specific digest<-database dependencies (first few lines)
# via make dependency combination rules
#
# baseline schema:
# key(particle, replicate, year), seronegative + imm1-imm4 (all %), s, c.s (ints)
#
# intervention schema, same, but with:
# key(vc, vac, vc_coverage, vaccine, catchup) added (on the left) to cols
#
# foi_% schema: same as baseline + intervention, w/ key(foi) added on the left
# lag_% schema: same as intervention, w/ key(vac_first) added on the left


# specific output database dependencies for each digest target
baseline.rds intervention.rds baseline.csv intervention.csv: $(REFDB)
foi_baseline.rds foi_intervention.rds foi_baseline.csv foi_intervention.csv: $(FOIDB)
lag_intervention.rds lag_intervention.csv: $(LAGDB)

baseline.sql intervention.sql foi_baseline.sql foi_intervention.sql lag_intervention.sql: sql.R projref.rda
	Rscript $^ $@

%.csv: %.sql
	sqlite3 -csv $(word 2,$^) < $< > $@

baseline.rds intervention.rds foi_baseline.rds foi_intervention.rds lag_intervention.rds: parse.R utils.R projref.rda
	Rscript $^ $@

ALLDIG := $(addsuffix .rds,baseline intervention foi_baseline foi_intervention lag_intervention)
# NB: make -j safe
digests: $(ALLDIG)
clean_digests:
	rm -f $(ALLDIG)

# EFFECTIVENESS CALCULATIONS

# specific input digest dependencies for each effectiveness target
effectiveness.rds: $(addsuffix .rds,baseline intervention)
foi_effectiveness.rds: $(addsuffix .rds,foi_baseline foi_intervention)
lag_effectiveness.rds: $(addsuffix .rds,baseline lag_intervention)
ALLEFF := $(addsuffix .rds,effectiveness foi_effectiveness lag_effectiveness)

# TODO document c = cumulative
# TODO rename ind = naive?
# TODO breaking comment bars in R?

# create data.table effectiveness computations
#
# basic (effectiveness.rds) schema:
# key(vc, vac, vc_coverage, vaccine, catchup, particle, replicate, year)
# bcases (baseline cases), icases (int. cases), c.bcases, c.icases (cum. cases of base and int.)
# eff ((bcases - icases)/bcases), c.eff ((c.bcases - c.icases)/c.bcases),
# logm (log(icases/bcases)), c.logm(log(c.icases/c.bcases))
#
# foi_% schema: same as basic, w/ key(foi) added on the left
# lag_% schema: same as basic, w/ key(vac_first) added on the left
effectiveness.rds foi_effectiveness.rds lag_effectiveness.rds: eff.R
	Rscript $^ $@

# make -j safe
effectiveness: $(ALLEFF)
clean_effectiveness:
	rm -f $(ALLEFF)

# COMBINATION EFFECTIVENESS CALCULATIONS
# combination effectiveness calcs compare naive combinations of interventions
# (naive == each intervention is acting independently)
# with observed combined outcomes
# schema:
# key'd according to underlying effectiveness, then
# combo.eff, c.combo.eff
# vec.eff, c.vec.eff, vac.eff, c.vac.eff
# ind.eff, c.ind.eff
comboeff.rds: effectiveness.rds
foi_comboeff.rds: foi_effectiveness.rds
ALLCMB := $(addsuffix .rds,comboeff foi_comboeff lag_comboeff)

comboeff.rds foi_comboeff.rds: comboeff.R
	Rscript $^ $@

# lag effectiveness has to be treated specially
lag_comboeff.rds: lag_comboeff.R lag_effectiveness.rds effectiveness.rds
	Rscript $^ $@

# NB: make -j safe
comboeff: $(ALLCMB)
clean_comboeff:
	rm -f $(ALLCMB)

## EFFECTIVENESS STATS CALCS
# collapses the particle*replicate keys for effectiveness results
# schema is key of underlying effectiveness / comboeff.rds +
# variable (the combo.eff non-key cols) +
# lo.lo (2.5%), lo (25%), med (50%), hi (75%), hi.hi (97.5%) quantiles

effstats.rds: comboeff.rds
foi_effstats.rds: foi_comboeff.rds
lag_effstats.rds: lag_comboeff.rds
ALLEFS := $(addsuffix .rds,effstats foi_effstats lag_effstats)

effstats.rds foi_effstats.rds lag_effstats.rds: effstats.R utils.R projref.rda
	Rscript $^ $@

# NB: make -j safe
effstat: $(ALLEFS)
clean_effstat:
	rm -f $(ALLEFS)

clean: clean_digests clean_effectiveness clean_comboeff clean_effstat
	rm projref.rda