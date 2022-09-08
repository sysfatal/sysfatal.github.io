---
title:      La próxima vez que te pregunten cada cuánto se cambian las contraseñas...
summary:    Frequent Password Changes Is a Bad Security Idea.
categories: blog
date:       Thu Dec 12 11:15:55 CET 2019
thumbnail:  passwords
image:      /figs/postit.jpg
layout:     post
thumbnail:  passwords
author:     e__soriano
tags:
 - passwords
 - contraseñas
---

<div class="share-page">
    Share this on &rarr;
    [<a href="https://twitter.com/intent/tweet?text={{ page.title }}&url={{ site.url }}{{ page.url }}&via=e__soriano&related=e__soriano" rel="nofollow" target="_blank" title="Share on Twitter">Twitter</a>]
    [<a href="https://facebook.com/sharer.php?u={{ site.url }}{{ page.url }}" rel="nofollow" target="_blank" title="Share on Facebook">Facebook</a>]
</div>
<br>

<center>
<figure class="image">
  <img src="figs/postit.jpg">
  <figcaption> Greek Minister head of intelligence's password post-it.</figcaption>
</figure>
</center>

Hay muchos _consultores de seguridad_ y _expertos_ que siguen repitiendo como
un loro algo parecido a esto, que os sonará:

>Hay que cambiar las contraseñas cada N meses, y es obligatorio
>establecer políticas que fuercen a los usuarios a ello.

No pasa un mes sin que alguien me diga que un _experto_ le ha obligado a seguir
esta política **obsoleta** y **dañina**.

Como siempre, todo depende del tipo de entorno y de los bienes que se intenten
asegurar. En la mayoría de los casos, esta práctica es muy poco recomendable:
invita a los usuarios a apuntar sus contraseñas (esos post-its!), elegir
contraseñas de poca calidad, rotarlas, etc.

Esto no lo digo yo (bueno, sí, pero otros también). Desde hace mucho tiempo,
autores como Bruce Schneier y Ross Anderson consideran que es una mala política.

El propio NIST recomienda no cambiar las credenciales si no hay indicios de que
hayan sido comprometidas, lo hace en
_2017 NIST Special Publication 800-63B, Authentication and Lifecycle Management_:

> Do not require that memorized secrets be changed arbitrarily (e.g., periodically)
> unless there is a user request or evidence of authenticator compromise.


Por si todavía necesitáis convencer a esos _expertos_ o a vuestros jefes:

* _"Frequent Password Changes Is a Bad Security Idea. I've been saying for
years that it's bad security advice, that it encourages poor passwords."_
[Bruce Scheneier](https://www.schneier.com/blog/archives/2016/08/frequent_passwo.html).

* _"One of the most pervasive and persistent design errors has been forcing users to change
passwords regularly. Indeed, this has almost become a religious war between security researchers and auditors."_
[Ross Anderson](https://www.cl.cam.ac.uk/~rja14/book.html).

* _"Why is it that the FTC is going around telling everyone to change their
passwords? Frequent password changes are the enemy of security"_
[Lorrie Cranor](https://www.ftc.gov/news-events/blogs/techftc/2016/03/time-rethink-mandatory-password-changes), Federal Trade Commission Chief Tech., Carnegie Mellon University.

* _"We quantify the security advantage of a password expiration policy, finding that the optimal benefit is relatively minor at best and questionable in light of overall cost"_
[Chiasson et al.](https://t.co/ek3UM6TWRp?amp=1), Quantifying the Security Advantage of Password Expiration Policies

* _"Password expiration is a dying concept: OUTDATED THREAT MODEL + BEHAVIORAL COST + INCREASING RISK"_
[Lance Spitzner](https://www.sans.org/security-awareness-training/blog/time-password-expiration-die), Director, SANS Security.

* _"Password Expiration Considered Harmful"_ [Rick Smith](https://cryptosmith.com/password-sanity/exp-harmful/)

* _“The more often users are forced to change passwords, the greater the overall vulnerability to attack”_
[Ciaran Martin](https://www.maytech.net/blog/why-regularly-changing-password-puts-you-at-risk-of-attack/), Head of the National Cyber Security Centre (NCSC), UK.

Si os quedan dudas sobre el asunto, [aquí](https://littlemaninmyhead.wordpress.com/2019/07/28/collection-of-references-on-why-password-policies-need-to-change/amp/)
podéis encontrar más información.

En el [Esquema Nacional de Seguridad](https://t.co/HkghesyoUj?amp=1) se dice:

>"Las contraseñas se cambiarán con una cierta periodicidad. Un año parece un tiempo razonable para su sustitución."

Parece que no se han querido mojar mucho. ¿_Parece razonable_ en qué contexto y para qué
credenciales?
Por un lado, para los _creyentes_  del cambio periódico, un año sería una
barbaridad (he escuchado casos en los que obligaban a cambiar **¡cada mes!**).
Por otro, como hemos visto, el NIST dice que no hay que cambiarlas de forma
periódica.

Parece que el sentido común se está imponiendo, y
hasta Microsoft [ha cambiado su política este mismo año](https://arstechnica.com/information-technology/2019/06/microsoft-says-mandatory-password-changing-is-ancient-and-obsolete/):

> There’s no question that the state of password security is problematic and has been for a long time. When humans pick their own passwords, too often they are easy to guess or predict. When humans are assigned or forced to create passwords that are hard to remember, too often they’ll write them down where others can see them. When humans are forced to change their passwords, too often they’ll make a small and predictable alteration to their existing passwords and/or forget their new passwords. When passwords or their corresponding hashes are stolen, it can be difficult at best to detect or restrict their unauthorized use.
>
> Recent scientific research calls into question the value of many long-standing password-security practices, such as password expiration policies, and points instead to better alternatives such as enforcing banned-password lists (a great example being Azure AD password protection) and multi-factor authentication. While we recommend these alternatives, they cannot be expressed or enforced with our recommended security configuration baselines, which are built on Windows’ built-in Group Policy settings and cannot include customer-specific values.

Seguro que en un mes lo vuelvo a escuchar XD

<sub><sup>
    <b>(cc) Enrique Soriano-Salvador</b>
    Algunos derechos reservados. Este trabajo se entrega bajo la licencia
    Creative Commons Reconocimiento - NoComercial - SinObraDerivada (by-nc-nd).
    Creative Commons, 559 Nathan Abbott Way, Stanford,
    California 94305, USA.
</sup></sub>
