# Renders cv.html to the PDF the site links to. Needs Google Chrome.
CHROME ?= /Applications/Google Chrome.app/Contents/MacOS/Google Chrome
../static/nikhil-acharya-prakash-cv.pdf: cv.html
	"$(CHROME)" --headless=new --disable-gpu --no-pdf-header-footer --virtual-time-budget=5000 --print-to-pdf=$@ file://$(CURDIR)/cv.html
