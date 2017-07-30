---
title: creando un blog con precios cuidados
date: 2017-07-08 12:08:11
tags: 
- Hexo
- Nodejs
- GitHub
- GitHub Pages
categories:
- Tecnología
- Blogs
---

Hola! Para empezar con ésto, te voy a contar como hice para armar el blog en una tarde y sin gastar nada más que un poco de tiempo.

## Opciones al momento de armar el blog

Por supuesto, lo primero que tenemos que hacer antes de arrancar con algo es analizar las herramientas que tenemos disponibles, sino terminamos golpeando un clavo con un destornillador... Cuando pensé en arrancar con ésto, lo primero que hice fue empezar a mirar un poco las tecnologías con las que estaban hechas los blogs que frecuento. De ese análisis, con las opciones que me quedé fueron básicamente:

- Blogspot
- GitHub Pages
- Ghost
- Wordpress + Hosting Propio

Ghost y Hostear una solución estilo Wordpress las descarté por ahora, por el simple hecho de son soluciones pagas y/o requieren tiempo en mantenimiento; y de las opciones restantes, opté por GitHub Pages porque me pareció la solución más flexible para lo que tenía en mente. Blogspot no me parece una mala opción para armar un blog de forma sencilla, rápida y de bajo costo, pero con GitHub Pages podía "meter un poco más de mano", así que me subí a ese tren.

## Generando contenido

La solución que ofrece GitHub Pages se basa en una idea bastante sencilla: "tirame HTML a un repositorio y yo me encargo del resto", lo cual es excelente! Peeeero, ponernos a generar a mano un blog escribiendo en HTML puro tampoco tenía demasiado sentido, ya que el tiempo vale oro, y dijimos que ésto iba a ser con precios cuidados! Acá es donde entra en juego el otro protagonista de la historia: [Hexo](https://hexo.io).

Hexo es básicamente una herramienta que permite tomar (o crear) un template base que indica "como" se va a generar el HTML final a mostrar, y una vez configurado, simplemente centrarnos en escribir con una sintaxis sencilla en texto plano (utilizando [Markdown](https://daringfireball.net/projects/markdown/)) nuestro contenido. Ésto tiene una gran ventaja para los que no nos llevamos del todo bien con el diseño, y es que sólo nos tenemos que enfocar en escribir, nada de pensar en HTML, CSS, y demás.

En la [documentación oficial de GitHub Pages](https://help.github.com/articles/using-jekyll-as-a-static-site-generator-with-github-pages/) indican como sugerencia utilizar [Jekyll](https://jekyllrb.com/), que es una herramienta muy similar a Hexo pero hecha en Ruby. Como no tengo demasiada experiencia con Ruby, y en mi equipo tenía instalado casi todo lo necesario para usar Hexo (básicamente se necesita Node, NPM, un editor de textos y no mucho más), directamente opté por ésta última herramienta.

## Diseño y Templates

Como ya dije más arriba, el diseño no es lo mío, por lo tanto fue una característica más a tener en cuenta al momento de decidir usar Hexo. Dentro de la [web oficial](https://hexo.io/themes/) hay varios themes libres de muy buena calidad. El que estoy utilizando actualmente es [Cactus Dark](https://probberechts.github.io/cactus-dark/), con unas modificaciones mínimas, principalmente en lo que tiene que ver con el idioma.

## Algunos detalles más...

Por último, como comentario, GitHub Pages hace un deploy automático de la última versión del branch *master* de nuestro repositorio en cada commit. Es decir, lo único que tenemos que hacer para actualizar el blog es hacer un push de los archivos resultantes generados por Hexo (hasta tiene un comando para hacer ésto por nosotros :) ) y después de unos minutos ya vamos a estar viendo los cambios reflejados en la web.

Dado que el branch *master* se usa para la parte "pública" del sitio, no se van a versionar en dicho branch los fuentes a partir de los cuales Hexo genera el contenido. Para solucionar esto, lo que hice fue crear un branch más en el repositorio totalmente independiente de master, donde se guardan los fuentes propiamente dichos. Debemos tener en cuenta que los branches *master* y *source* (como lo llamé en mi caso) nunca se van a unir entre si ya que su contenido difiere totalmente, es decir, en la práctica actuarían como dos "repositorios" totalmente independientes.


Hasta acá llego por hoy, en el próximo post voy a hacer un pequeño resumen de cómo se usa Hexo y cómo es el proceso de exportar y subir el sitio. Espero que te haya resultado útil!

Saludos y Happy Blogging!
