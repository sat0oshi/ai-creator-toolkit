---
name: "kreasite"
description: "Recrée un site web impressionnant (Three.js, WebGL, motion) à partir d'une vidéo, capture ou lien de référence, dans la DA et pour le projet de l'utilisateur."
---

# KreaSite

L'utilisateur montre un site qui l'impressionne (vidéo d'écran, capture, lien). Tu le décortiques et tu construis un site du même niveau, avec la même mécanique et la même sensation, mais dans SA direction artistique et pour SON projet.

On reprend la technique, la grammaire de caméra, le rythme et l'interface. On ne copie jamais les textes, logos, noms, marques, images ou modèles 3D de la référence. Le résultat doit être reconnaissable comme "du même niveau", pas comme une copie.

Déroulé : 1. analyser, 2. récupérer le projet et la DA, 3. transposer, 4. construire, 5. tester, 6. livrer. Crée une liste de tâches avec ces étapes dès le début.

---

## 1. Analyser la référence

**Vidéo** (mp4, mov, webm) : extrais des images et regarde-les en planches.

```bash
mkdir -p ref && cd ref
ffprobe -v error -show_entries format=duration -of default=nw=1 "$VIDEO"
# 1 image toutes les 2 s (toutes les 4 s si la vidéo dépasse 60 s)
ffmpeg -v error -i "$VIDEO" -vf "fps=1/2,scale=960:-1" f_%02d.jpg
# planches de 8 images
ffmpeg -v error -y -i f_%02d.jpg -vf "tile=2x4" sheet_%d.jpg
```
Lis chaque planche avec Read. Si une transition t'intrigue, extrais 3 ou 4 images autour de ce moment avec `-ss`.

**Capture** : lis-la directement.

**Lien** : WebFetch pour la structure et les textes. Si un outil navigateur est dispo, prends une capture. Si le site est injoignable, demande une vidéo d'écran.

Écris pour toi une fiche de décodage courte (pas dans la réponse) :
- Concept en une phrase (ex : "trou noir au-dessus d'un décor, deux mondes qu'on switche").
- Technique probable : shader plein écran / raymarching, particules, InstancedMesh (herbe, fleurs), caméra pilotée au scroll, modèle GLTF, vidéo de fond, simple CSS animé...
- Scènes ou sections, et comment on passe de l'une à l'autre.
- Interactions : drag, molette, scroll, hover, clavier, boutons.
- Interface : où sont placés les éléments (coins, HUD, menu, CTA), typo (serif, mono, display), tailles.
- Palette, lumière, grain, flou, vignettage.
- Le moment "wow" : ce qui rend le site impressionnant. C'est ça qu'il faut absolument reproduire.
- Ce qui vient d'assets externes (modèles 3D, photos, vidéos) et qu'il faudra recréer en procédural.

---

## 2. Récupérer le projet et la DA

Cherche d'abord dans la conversation et la mémoire : nom, projet, offre, cible, couleurs, polices, langue, liens, chiffres réels, réalisations.

S'il manque l'essentiel, pose UNE seule série de questions (AskUserQuestion, 3 questions max) :
1. Le projet et le but du site (vitrine perso, offre, produit, portfolio...).
2. La DA : couleurs et polices, ou "propose-moi".
3. Les contenus : liens (réseaux, offre, contact), chiffres, réalisations.

Si l'utilisateur a dit "fais" ou ne répond pas, pars sur la meilleure lecture et dis-le en une ligne.

Règles de contenu :
- N'invente jamais de chiffres, prix, récompenses, clients ou URL.
- Un lien inconnu pointe vers une page connue de l'utilisateur (son site, son profil) et tu le signales à la fin.
- Textes dans la langue de l'utilisateur, courts, naturels, sans jargon marketing.

---

## 3. Transposer

Fais une table de correspondance, élément par élément : référence → version de l'utilisateur.

- **On garde** : la grammaire de caméra, la composition, le rythme, la structure de l'interface, les interactions, le moment wow.
- **On remplace** : le sujet et le décor par l'univers de l'utilisateur (sa ville, son métier, ses objets, sa silhouette), la palette par sa DA, les textes par son contenu.
- Ajoute au moins un détail que seul ce projet peut avoir : coordonnées GPS de sa ville, ses vrais chiffres, un objet signature (casquette, outil, produit).
- Si la référence a plusieurs scènes, garde le même nombre et donne-leur des noms tirés de la DA (ex : "Or" et "Fuchsia").

Exemple réel : ferme américaine + pickup sous un trou noir doré → villa moderne sur un cap de la Côte d'Azur avec mer et palmiers. Robot dans un champ violet → silhouette de dos en casquette à l'envers dans un champ fuchsia.

---

## 4. Construire

### Choisir la stack
- Référence 3D / WebGL (scène, particules, shader) → **Three.js 0.160.0** en modules via importmap.
- Référence surtout scroll et motion 2D → HTML/CSS + **GSAP 3.12.5** (cdnjs, avec ScrollTrigger).
- Fond génératif simple → canvas 2D ou un seul shader plein écran.

### Contraintes
- Un seul fichier HTML : CSS et JS inline.
- Scripts uniquement depuis cdnjs.cloudflare.com, cdn.jsdelivr.net/npm, unpkg.com, avec version exacte. Polices uniquement Google Fonts, avec une vraie pile de secours.
- Aucune image, vidéo ou modèle externe : tout est procédural (Box, Capsule, Lathe, Tube, Sphere, InstancedMesh, shaders). Si l'utilisateur fournit ses propres images, intègre-les en data URI (16 Mo max au total).
- Pas de iframe (YouTube, maps) : des liens à la place.

### Squelette Three.js qui marche

```html
<script type="importmap">
{ "imports": {
  "three": "https://cdn.jsdelivr.net/npm/three@0.160.0/build/three.module.js",
  "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.160.0/examples/jsm/"
} }
</script>
<script type="module">
import * as THREE from 'three';
import { EffectComposer } from 'three/addons/postprocessing/EffectComposer.js';
import { RenderPass } from 'three/addons/postprocessing/RenderPass.js';
import { UnrealBloomPass } from 'three/addons/postprocessing/UnrealBloomPass.js';
import { ShaderPass } from 'three/addons/postprocessing/ShaderPass.js';
import { OutputPass } from 'three/addons/postprocessing/OutputPass.js';
</script>
```

Organisation :
- `WORLDS = { a: {...}, b: {...} }` : toute la config d'une scène (position caméra, pitch, focale, couleurs du ciel, brouillard, paramètres du sujet, bloom). Une fonction `setWorld(key)` copie ces valeurs dans les uniforms et affiche le bon `THREE.Group`.
- Uniforms partagés : un objet `U = { uTime, uSkyTop, uSkyHor, uBand, uGlow, uLightDir, uFogDen, uFogTint, ... }` étalé dans chaque ShaderMaterial (`uniforms: { ...U, ...propres }`). Une seule mise à jour sert à tous les matériaux.
- Un bloc GLSL `COMMON` inclus partout : hash, value noise, fbm, `skyBase(dir)` et `applyFog(col, worldPos)`. Le brouillard tend vers la couleur du ciel à l'horizon, donc le sol se fond dans le ciel sans couture.

```glsl
vec3 skyBase(vec3 d){
  vec3 c = mix(uSkyHor, uSkyTop, smoothstep(0.0, 0.55, max(d.y, 0.0)));
  c += uBand * exp(-abs(d.y) * uBandK);                 // bande à l'horizon
  float az = max(dot(normalize(d.xz + 1e-5), normalize(uLightDir.xz + 1e-5)), 0.0);
  c += uGlow * pow(az, uGlowSpread) * exp(-abs(d.y) * uGlowK); // halo vers la source, décroissance à part
  return c;
}
vec3 applyFog(vec3 col, vec3 wp){
  vec3 dv = wp - cameraPosition; float dist = length(dv); vec3 dir = dv / dist;
  float f = 1.0 - exp(-dist * uFogDen);
  col = mix(col, uFogTint, f);
  return mix(col, skyBase(normalize(vec3(dir.x, 0.0, dir.z))), f * f * f);
}
```

- **Ciel** : un quad plein écran (`PlaneGeometry(2,2)`, `gl_Position = vec4(position.xy, 0.9999, 1.0)`, `depthTest:false`, `depthWrite:false`, `renderOrder:-1000`, `frustumCulled:false`). Direction du rayon dans le fragment : `vec4 vp = uInvProj * vec4(vNdc,1,1); vp /= vp.w; vec3 d = normalize((uCamWorld * vec4(normalize(vp.xyz),0)).xyz);` avec `uInvProj = camera.projectionMatrixInverse` et `uCamWorld = camera.matrixWorld` passés par référence.
- **Sujet céleste** (trou noir, planète, soleil, portail) : raymarché dans le shader du ciel, dans un repère local placé loin de la caméra (`ro = (cameraPosition - pos) / scale`). Pour un trou noir : `v += -1.5 * h2 * p / pow(r, 5.0) * dt` avec `h2 = |cross(p, v)|²`, disque détecté au changement de signe de `p.y`, test d'intersection avec une sphère englobante pour sauter les rayons qui ne passent pas près.
- **Végétation / foule / particules** : `InstancedMesh` + ShaderMaterial (`instanceMatrix` est déclaré automatiquement), vent dans le vertex shader, attributs par instance via `InstancedBufferAttribute`.
- **Bâtiments, personnages** : primitives assemblées. Personnage crédible = torse en `LatheGeometry` aplati (profil hanches, taille, épaules), bras collés au corps, tête un peu petite. Matériau silhouette : base presque noire + liseré (fresnel) côté lumière.
- **Post-traitement** : RenderPass → UnrealBloomPass → OutputPass → ShaderPass final (vignette, grain, aberration chromatique très légère, courbe en S, `uFade` pour les transitions).
- **Caméra** : `rotation.order = 'YXZ'`, drag = yaw/pitch amortis, molette et pincement = focale (`camera.setFocalLength(mm)`), légère respiration et parallaxe souris. En portrait, élargis : `eff = aspect < 1.25 ? mm * Math.max(aspect / 1.55, 0.5) : mm` et recentre le yaw sur le sujet.
- **Intro** : la caméra descend vers sa position pendant 4 à 5 s pendant que le nom de l'utilisateur, en très grand, s'efface.
- **Interface** : reprends le placement de la référence (coins, identité en haut à gauche, outils en haut à droite, règle de focale, télémétrie avec de vraies valeurs calculées, switch des scènes en bas). Ajoute un bouton qui ouvre un panneau : bio, chiffres réels, projets avec liens, réseaux, un CTA principal.
- **Perf** : pixel ratio plafonné (1.35 desktop, 1.25 mobile), moins d'instances sur mobile, qualité adaptative (si FPS < 42 pendant 2 s, baisse le pixel ratio de 15 %), `renderer.compile()` des deux scènes au démarrage.
- **Robustesse** : message propre si WebGL échoue, `prefers-reduced-motion` respecté, boutons avec `aria-label`, focus visible.
- **Hooks de test** : `window.__SKIP_INTRO` lu au démarrage, et `window.__k = { setWorld, bloom, ... }` exposé.

### Pièges déjà rencontrés (à lire avant de régler les couleurs)
1. **Les valeurs des shaders sont linéaires.** Elles passent par ACES puis sRGB : 0,05 linéaire sort déjà gris clair à l'écran. Ordres de grandeur : fonds 0,0005 à 0,03, surfaces sombres 0,001 à 0,01, zones lumineuses 0,5 à 3. Un premier rendu tout blanc veut dire valeurs trop hautes, pas un bug.
2. **Bloom** : threshold 1,0 à 1,1, strength 0,25 à 0,45, radius 0,3 environ, réglé par scène. Un gros objet très lumineux avec un grand radius délave toute l'image en gris.
3. **Halos accumulés** (lueur volumétrique, anneau) : n'ajoute la lueur qu'aux rayons concernés (ex : rayons qui s'échappent, via leur distance minimale), sinon l'intérieur qui doit rester noir se remplit.
4. **Lueur d'horizon** : donne-lui sa propre décroissance verticale (`uGlowK`), sinon tout le ciel derrière le sujet s'éclaire.
5. **Sprites de bokeh près du sol** : `depthTest:false`, sinon ils sont coupés en deux.
6. **Échelle** : vérifie la taille angulaire. Un bâtiment de 10 m à 40 m avec une focale de 49 mm prend un tiers de l'écran. Un personnage à 6 m remplit le cadre.
7. **Chiffres dans le panneau** : `white-space: nowrap` et taille en `clamp()` pour éviter les retours à la ligne sur mobile.

---

## 5. Tester

Si un shell avec Chromium est dispo, teste en vrai avant de livrer :

```bash
mkdir -p site && cd site && npm init -y >/dev/null && npm install three@0.160.0 playwright
```

Script de capture (adapte `executablePath` au Chromium présent, par exemple `/opt/pw-browsers/chromium`) :

```js
import { chromium } from './node_modules/playwright/index.mjs';
import fs from 'fs'; import path from 'path';
const html = fs.readFileSync('site.html', 'utf8');
const doc = `<!doctype html><html><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover"></head><body>${html}</body></html>`;
const browser = await chromium.launch({ executablePath: process.env.CHROME, args: ['--use-angle=swiftshader', '--enable-unsafe-swiftshader', '--ignore-gpu-blocklist'] });
for (const [w, h, tag] of [[960, 540, 'desk'], [390, 844, 'mob']]) {
  const page = await browser.newPage({ viewport: { width: w, height: h } });
  page.on('pageerror', e => console.log('pageerror:', e.message));
  await page.route('**/*', r => {
    const u = r.request().url();
    if (u.startsWith('http://local/')) return r.fulfill({ body: doc, contentType: 'text/html' });
    const m = u.match(/three@0\.160\.0\/(.*)$/);
    if (m) return r.fulfill({ body: fs.readFileSync(path.join('node_modules/three', m[1])), contentType: 'application/javascript' });
    return r.abort();
  });
  await page.addInitScript(() => { window.__SKIP_INTRO = true; });
  await page.goto('http://local/');
  await page.waitForTimeout(5000); await page.screenshot({ path: `${tag}_a.png` });
  await page.evaluate(() => window.__k.setWorld('b'));
  await page.waitForTimeout(5000); await page.screenshot({ path: `${tag}_b.png` });
}
await browser.close();
```

- Regarde chaque capture. Compare-la aux images de la référence : exposition, cadrage, taille du sujet, lisibilité de l'interface.
- Compte 3 à 5 passes de réglage. La première est presque toujours surexposée.
- Pour isoler un problème de lumière, refais une capture avec `bloom.strength = 0`.
- Les FPS affichés en rendu logiciel ne veulent rien dire. Les Google Fonts ne chargent pas dans ce test : c'est normal.
- Teste aussi le panneau ouvert et le switch de scène.

Sans shell : relis le code ligne à ligne (GLSL surtout) avant de livrer, et dis que le rendu n'a pas pu être vérifié.

---

## 6. Livrer

- Si l'outil Artifact est dispo, publie la page comme artifact (suis ses règles : `<title>` court au nom du projet, pas de balises html/head/body, scripts des CDN autorisés). Sinon, envoie le fichier.
- Envoie aussi le fichier HTML. Pour un hébergement sur leur propre domaine, ajoute en tête `<!doctype html><html lang="fr"><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">` et ferme les balises, sinon la page tourne en mode quirks.
- Message final court, sans récap des étapes :
  - ce que montre chaque scène et le moment wow ;
  - comment on interagit ;
  - ce qu'il faut vérifier : liens provisoires, perf non mesurée sur une vraie carte graphique, page privée tant qu'elle n'est pas partagée ;
  - une seule proposition de suite si elle a du sens.