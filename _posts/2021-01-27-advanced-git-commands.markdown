---
layout: post
title:  "Comandos avanzados en GIT"
date:   2021-01-27 09:00:00 -0300
permalink: comandos-avanzados-en-git
---
GIT es una herramienta imprescindible a la hora de trabajar como desarrollador. Este post va orientado a gente que ya tiene una noción básica de la herramienta, con el objetivo de optimizar su uso y conocer útiles comandos que nos pueden sacar de aprietes.

# git stash [📖](https://git-scm.com/docs/git-stash){:target="_blank"}
Si solo pudiera elegir un comando de git, definitivamente sería `git stash`. Una vez que lo descubrí, mi workflow cambió por completo. La traducción de stash hace referencia a reserva, lo que te permite este comando es guardar todos los cambios que veníamos haciendo en staging sin necesidad de forzar un commit para poder dedicarnos a otra cosa. 

Un escenario bastante usual en el que hacía uso de este comando es cuando estaba trabajando en una feature particular, pero surgía algún bug en producción y tenía que dejar lo que estaba haciendo para tirar un hotfix en la codebase. No iba a deshacer todo lo que venía haciendo, por lo que hacía un `git stash` (o `git stash -u` en el caso de que untracked files), abría una nueva branch para solucionar el bug, y luego retornaba a lo que estaba haciendo en la rama de la feature con un `git stash pop`. En el caso de que los cambios stasheados no sean útiles se pueden eliminar con `git stash drop`. Además podemos tener más de un stash guardado, por lo que podemos ver la cantidad de stashes con `git stash list`.

<img src="https://res.cloudinary.com/dd28ghazj/image/upload/v1611013931/santiagollapur.com/git-stash_mkh9bd.png">

Para entender más en profundidad recomiendo leer el siguiente [blogpost](https://www.atlassian.com/es/git/tutorials/saving-changes/git-stash){:target="_blank"}.

# git uncommit (reset) [📖](https://git-scm.com/docs/git-reset){:target="_blank"}
Cuantas veces nos pasó de hacer un commit localmente y nos olvidamos de modificar algo demasiado diminuto como para hacer otro nuevo commit. Con `git reset --soft HEAD^` van a poder revertir ese commit y pasarlo a staging para agregar los cambios imprevistos. Al mismo también le puse alias: `git config --global alias.uncommit 'reset --soft HEAD^'`.

<img src="https://res.cloudinary.com/dd28ghazj/image/upload/v1611602695/santiagollapur.com/advanced-git-commands/git_uncommit_skzhvv.png">

# git unstage (reset) [📖](https://git-scm.com/docs/git-reset){:target="_blank"}
Similar al de uncommit pero en staging, nos puede pasar que hicimos `git add` con algún que otro archivo que no queremos commitear ahora, por lo que necesitamos sacarlo de staging con `git reset nombredelarchivo.txt`. Al igual que los otros ejemplos, lo uso con alias `git config --global alias.unstage 'reset'` ya que el reset me parece que queda poco claro.

<img src="https://res.cloudinary.com/dd28ghazj/image/upload/v1611605200/santiagollapur.com/advanced-git-commands/git_unstage_e4qjeq.png">

# git checkout [📖](https://git-scm.com/docs/git-checkout){:target="_blank"}
Probablemente ya conozcan este comando para moverse de branches, pero también tiene otro caso de uso que es para cuando hicimos cambios en un archivo, pero nos dimos cuenta que queremos deshacerlos, en vez de estar borrando linea por linea con un simple `git checkout nombredelarchivo.txt` se revierten los cambios. Hay que tener cuidado con este comando porque una vez que se ejecuta el checkout, no se pueden recuperar los cambios hechos.

<img src="https://res.cloudinary.com/dd28ghazj/image/upload/v1611702041/santiagollapur.com/advanced-git-commands/git_checkout_r5md8x.png">

# git sla / gloga [📖](https://git-scm.com/docs/git-log){:target="_blank"}
Ambos comandos son inexistentes, son abreviaciones que cree a raíz de `git log --oneline --decorate --graph --all`, que vendría a ser una manera de ver todo el commit history de una forma más agradable, para entender cómo fueron mergeadas las ramas en el caso de que no seamos los únicos trabajando en un proyecto. Para crear el alias deberíamos escribir lo siguiente `git config --global alias.sla 'log --oneline --decorate --graph --all'`. sla hace referencia por "short log all", pero cada uno puede modificar el alias a su gusto. Por otro lado, si queremos abreviar más todavía esta función, podríamos crear un alias (en mi caso uso Oh My Zsh para crear esto) como `alias gloga='git log --oneline --decorate --graph --all'`.

<img src="https://res.cloudinary.com/dd28ghazj/image/upload/v1611591990/santiagollapur.com/advanced-git-commands/git_log_c3wsjg.png">

# git blame [📖](https://git-scm.com/docs/git-blame){:target="_blank"}
`git blame nombredelarchivo.txt` sirve para ver que linea de un archivo fue modificada, la fecha en que se modificó, el autor de ese cambio, y el commit el cual se hizo. No suelo utilizar mucho este comando porque le agregué a mi editor de texto un plugin para ya verlo desde ahí sin tener la necesidad de escribir en la terminal ([VScode](https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens#current-line-blame-), [Sublime Text](https://packagecontrol.io/packages/Git%20blame)). 

Un escenario de uso se me dió cuando tenía que refactorizar unos métodos, en donde el código no estaba muy declarado o no llegaba a entender en qué contexto se escribió, por lo que me fijaba con el blame, y le consultaba a aquella persona que trabajó previamente en eso.

<img src="https://res.cloudinary.com/dd28ghazj/image/upload/v1611596387/santiagollapur.com/advanced-git-commands/git_blame_ipwfrw.png">

# Algunos extras:
- `git diff --cached`: Sirve para ver el diff de los archivos que estén en staging.
- `git add . && git commit --amend --no-edit`: Agrega cambios que te olvidaste de agregar al último commit.
- `git show hash-del-commit`: muestra los cambios que hizo una persona en determinado commit
- `git cherry-pick hash-del-commit`: Este comando nos ofrece la posibilidad de agregar un commit ya hecho de cualquier otra rama a la actual que estemos parados.
- `git reflog`: muestra todas las operaciones locales que hiciste en determinado proyecto.
- `git tag nombredeltag`: se le asocia un tag al próximo commit que hagas, esta bueno para cuando se hacen releases de nuevas versiones de app u otros sucesos importantes.
- HEAD: es la referencia al último commit del branch en el que estamos parados.

Si este post te resultó interesante y queres investigar más del tema, recomiendo hacer el curso gratuito de [Mastering Git](https://thoughtbot.com/upcase/mastering-git){:target="_blank"} de thoughtbot, y si te interesa profundizar en como funciona git, leer [The Git Object Model](https://shafiul.github.io/gitbook/1_the_git_object_model.html)
