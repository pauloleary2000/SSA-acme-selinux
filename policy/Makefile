.RECIPEPREFIX := >
MODULES := $(basename $(notdir $(wildcard *.te)))
all: $(addprefix build/,$(addsuffix .pp,$(MODULES)))
build/%.pp: %.te
> mkdir -p build
> checkmodule -M -m -o build/$*.mod $<
> semodule_package -o $@ -m build/$*.mod
> sha256sum $@ > $@.sha256
clean:
> rm -rf build
