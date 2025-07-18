run:
	docker run --rm -itd --name texlive --volume $(PWD)/docs:/workdir paperist/alpine-texlive-ja:latest sh -c "latexmk -pvc css.tex"

attach:
	docker exec -it texlive /bin/sh

stop:
	docker stop texlive