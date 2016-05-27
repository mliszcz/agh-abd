
BUILDDIR = build
DISTDIR = dist

%.pdf: %.tex $(BUILDDIR) $(DISTDIR)
	latexmk -cd- -pdf -interaction=batchmode -output-directory=$(BUILDDIR) $<
	cp $(BUILDDIR)/$@ $(DISTDIR)/$@

$(BUILDDIR):
	mkdir -p $(BUILDDIR)

$(DISTDIR):
	mkdir -p $(DISTDIR)

all: $(patsubst %.tex, %.pdf, $(wildcard *.tex))

clean:
	rm -rf $(BUILDDIR)

.PHONY: all clean
