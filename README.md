Use of DANE to improve security for identity federations
========================================================
This Github project is one of two repositories used for our Bachelor Thesis work.
This repository (BachelorThesis) has our report in LaTeX as well as some installation and configuration instructions for the different software we have used. The other github project (shibIdPextension) has our DANE implementation for the Shibboleth Identity Provider.

### Our thesis and installation/configuration instructions
https://github.com/christofferholmstedt/BachelorThesis

To get our report in a readable PDF we have used "pdflatex" on Ubuntu 11.04 and 12.04. It should be enough to download package "texlive". After cloning this project the following commands should do it.

    sudo apt-get update
    sudo apt-get install texlive
    cd /path/to/project/directory/thesis
    pdflatex thesis.tex
    bibtex thesis.tex
    pdflatex thesis.tex
    pdflatex thesis.tex

### Our DANE implementation
https://github.com/christofferholmstedt/shibIdPextension

### Best regards
Sophia Bergendahl and Christoffer Holmstedt, May 2012

