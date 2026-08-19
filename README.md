# Auto Škola — Webapp za učenje pitanja

Interaktivna web aplikacija za učenje teorijskih pitanja za vozački ispit u Srbiji.

## Kako pokrenuti lokalno

```bash
python -m http.server 8000
# Otvori http://localhost:8000
```

## Funkcionalnosti

### Tab: Učenje
- **Accordion prikaz** — kategorije i podoblasti, otvara se klik-om
- **Interaktivni odgovori** — klikni na odgovor, odmah dobiješ tačno/netačno feedback
- Odgovori su uvek **izmešani** (stabilan random po pitanju)
- Kartica postaje **zelena** (tačno) ili **crvena** (netačno) — žuta ako je označena
- **Multi-choice** podrška — pitanja sa više tačnih odgovora
- **Označavanje pitanja** zastavicom — čuva se u `localStorage`
- **Pretraga** po tekstu pitanja i odgovorima
- **Fokus Mod** — prolaz kroz pitanja jedno po jedno (modal), pamti poziciju
- Fokus može da se otvori od **bilo kog pitanja** (ikonica nišana pored svakog pitanja)

### Tab: Saobraćajni Znaci
- Grid prikaz svih pitanja sa slikama znakova
- **Filter** — samo saobraćajni znaci / sve slike
- **Kviz Znakova** — prolaz kroz znake sa interaktivnim odgovorima i skorom

### Opšte
- **Fullscreen slike** — klik na sliku otvara lightbox
- Tastatura: `←→` navigacija u modalima, `Esc` zatvara, `Space` prikaži rešenje

## Tehnologije

- Vanilla JS (bez framework-a)
- Tailwind CSS (CDN)
- FontAwesome ikone (CDN)
- Cloudinary (hosting slika)

## Struktura podataka

```
autoskola_baza.json   — baza svih pitanja
index.html            — kompletna aplikacija (single-file)
```
