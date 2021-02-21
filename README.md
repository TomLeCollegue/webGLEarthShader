# `Earth WebGL` 
###   📱 A website in tree.js that show the Earth with real day and night time ! 
###   See the result : [![Website](https://img.shields.io/static/v1?label=WebGlEarth:&message=v1.0&color=blue)](https://tomlecollegue.github.io/webGLEarthShader/noTimer.html)
###   Wallpaper Version : [![Website](https://img.shields.io/static/v1?label=WallPaper:&message=v1.0&color=blue)](http://89.87.13.28:8181/nextcloud/index.php/s/EKzYGMDnD7bdHr2)
<img src="https://github.com/TomLeCollegue/webGLEarthShader/blob/master/earth2.gif?raw=true" alt="drawing" width=100% align="center" />


## `Explication TP`

### Le code en question est visible sur `noTimer.html`

Je voulais faire une Terre avec la partie sombre avec les lumieres des villes et la partie éclairée avec avec la texture de la Terre de base. 
De base dans Three.js, il y a quelques Materiels par default avec des shaders pré-fait, avec la possibilité de mettre une map, une normalMap, une spacular map, alphaMap, etc...

Pour mon besoin il me fallait la possibilité de mettre une texture different en fonction de l'eclairage sur la sphère, ce qui n'est pas possible avec les shaders de base de Three.js. la solution est donc de faire un material custom pour la terre.

Three.js offre cette possibilité avec les ShadersMaterial().
Pour créer un Material custom Three.js nous demande naturellement : 
  * le "Vextex shader"
  * le "Fragment Shader" 
  * les "uniforms" qui seront les données utilisées par les shaders (direction de la lumieres, textures Map ... )
 
Pour commencer on calcule dans le vertex Shader vSunDir : la direction du soleil : 
```glsl
vSunDir = mat3(viewMatrix) * sunDirection;
```

On calcule dans le vertex Shader vNormal : la normal du polygone : 
```glsl
vNormal = normalMatrix * normal;
```

Dans le fragment Shader, on regarde si la surface est face au soleil ou non : 
```glsl
float cosineAngleSunToNormal = dot(normalize(vNormal), normalize(vSunDir));
```
--> Cela nous donne un resultat entre -1 et 1, si le nombre est negatif on est dans l'ombre, sinon on est eclairé

On modifie ca pour nous donne un entre 0 et 1 pour finir sur un mix entre les deux texture de nuit et de jour : 
```glsl
vec3 color = mix( nightColor, dayColor, mixAmount );
```

#### Avec cela nous avons bien une Terre avec une moitié avec la texture de nuit et une autre avec la texture de jour !
### Mais cela n'etait pas beau car on a une texture trop plate ! on rajoute donc une normalMap à tout ca.

Pour cela il nous faut calculer la tangente (vTangent) et la bitengente (vBitangent) :
```glsl
vec3 tangents;

vec3 c1 = cross(vNormal, vec3(0.0, 0.0, 1.0));
vec3 c2 = cross(vNormal, vec3(0.0, 1.0, 0.0));

if (length(c1)>length(c2))
{
    tangents = c1;
}   
else
{
    tangents = c2;
}
vTangent = normalize(tangents);

vBitangent = normalize( cross( vNormal, vTangent ));
```
Pour afficher la normale map on se sert de la direction de la lumiere, or on veut que la normal map soit aussi là dans l'obscurité, on selectionne donc la direction de la lumière vLL en fonction de l'orientation : si on est face au soleil on prend la direction du soleil, sinon on prend la direction opposé.
```glsl
vec3 vLL;
if(cosineAngleSunToNormal > 0.0){vLL = vSunDir;}else{vLL = -vSunDir;}
```
On calcul la couleur de la normale avec vLL, vNormal, vTangent, vBitangent et la normalMap :
```glsl
mat3 tsb = mat3( normalize( vTangent ), normalize( vBitangent ), normalize( vNormal ) );
vec3 finalNormal = tsb * normalTex;
float ndotl = dot (normalize (vLL), normalize (finalNormal));
//float ndotl = dot (normalize (vSunDir), normalize (finalNormal));

vec3 colorNormal = vec3( ndotl, ndotl, ndotl);
```

Enfin, On mixe la couleur de la normal avec la couleur de base, on augmente la luminosité pour pas que cela soit trop sombre (x1,6), et on l'affiche !!
```glsl
color.x = color.x * colorNormal.x * 1.6;
color.y = color.y * colorNormal.y * 1.6;
color.z = color.z * colorNormal.z * 1.6;

gl_FragColor = vec4( color, 1.0 );
```
#### Notre Terre est maintenant toute belle !

### Maintenant il faut l'orienter en fonction de la saison et de l'heure de la journée.
Cette partie est un petit peu approximative mais au final cela est plutot credible !

on met la terre dans un groupe, le groupe sera incliné de PI * 0.13 qui est l'inclinaison par rapport a l'axe de la terre.
cela nous permet de faire la rotation sur elle meme de la planete en 24h, mais aussi modifier la rotation en fonction de la saison.

On part du fait que :
 * à minuit le soleil nous oppose au maximum.
 * le 21 decembre, le pole nord est le plus eloigné du soleil de part la rotation de la Terre. 

Au lancement du site on met la terre tel qu'il soit minuit le 21 decembre.

pour calculer la saison, on calcule la difference de jour en le 21 decembre et la date du jour.
```js
days =  Math.floor(( today - Date.parse("12/21/2020") ) / 86400000); 
```
On sait que la terre revient au meme endroit par rapport au soleil en 365 jours
chaque jour on a donc une rotation de (2PI)/365, on fait donc : 
```js
//Aply rotation with the day for the season
 earthGroup.rotation.y += days * ((Math.PI * 2)/365);

 //Compensation Season on earth
 earthWCamera.rotation.y -=   days * ((Math.PI * 2)/365);
```

On fait la meme chose pour la rotation de la terre en elle meme en 24h
```js
var hour = today.getHours();
var minuteDecimal = today.getMinutes() / 60
hour = hour + minuteDecimal;
earthWCamera.rotation.y +=   hour * ((Math.PI * 2)/24);
```

Pour finir, On fait tourner la terre dans "animate" car on a une version fond d'ecran qui doit evoluer dans la journée.
```js
var angle = 0.1 * Math.PI * 2 * fracTime; // one turn per 10 second.
var angle1jour = angle / 6 / 60 / 24
var chosenAngle = angle1jour;

earthGroup.rotation.y += chosenAngle/365;
earthWCamera.rotation.y += chosenAngle;
cloud.rotation.y += angle/30
```















