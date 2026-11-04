# Deploying singulariteten.dk

The steps for launching the website on GitHub Pages at singulariteten.dk.

- **The website** is the `docs/` folder, and GitHub Pages publishes only that folder.
- **Everything else** stays off the website: these notes, `assets/social/`, `redirect-indisingulariteten/` and `checks/` (which git ignores). It is still visible on GitHub, because GitHub Pages on a free account needs a public repository. None of it is secret.

The commands below assume:

- the GitHub account `Gustavjiversen01`, which `gh` is logged in as on this machine;
- a repository named `singulariteten-site`.

## 0. Before launch day

1. **Register the domain.**
   - On 1 November 2026 neither singulariteten.dk nor indisingulariteten.dk had any DNS records, which suggests both are free. The registrar's own search is the real check.
   - A .dk domain is registered through a Danish registrar (Simply.com, One.com, DanDomain, Gigahost and others).
   - Expect to confirm your identity (MitID) with Punktum dk, the .dk registry.
   - Registering indisingulariteten.dk as well is optional (section 6).
2. **If singulariteten.dk cannot be had,** switch to the fallback domain and skip section 6:
   - set `DOMAIN = "indisingulariteten.dk"` in `~/Documents/Investment/output/final_k05/_build/launch.py`;
   - rebuild (see "Updating the site" below) and commit.
3. **Look at the site locally** one last time:

   ```
   cd ~/Documents/Investment/output/singulariteten-site
   python3 -m http.server -d docs 8000
   ```

   Open http://localhost:8000/ and http://localhost:8000/en/.

## 1. Create the repository and push

```
cd ~/Documents/Investment/output/singulariteten-site
git status                       # should say: nothing to commit
gh repo create singulariteten-site --public --source=. --remote=origin --push \
  --description "Ind i singulariteten / Into the Singularity, singulariteten.dk"
```

## 2. Verify the domain with GitHub (recommended)

This stops anyone else from serving a GitHub Pages site on the domain. It needs the domain to be registered.

1. On github.com, go to your profile **Settings > Pages > Add a domain**, and enter `singulariteten.dk`.
2. GitHub shows a TXT record. Its host is `_github-pages-challenge-gustavjiversen01` and its value is a code.
3. Add that record at the registrar (section 4).
4. Wait a few minutes, then press **Verify**.

## 3. Turn on GitHub Pages from docs/

```
gh api -X POST repos/Gustavjiversen01/singulariteten-site/pages \
  -f "source[branch]=main" -f "source[path]=/docs"
gh api -X PUT repos/Gustavjiversen01/singulariteten-site/pages -f cname=singulariteten.dk
gh api repos/Gustavjiversen01/singulariteten-site/pages          # shows status, cname and url
```

`docs/CNAME` already contains `singulariteten.dk`, and `docs/.nojekyll` makes GitHub serve the files exactly as they are.

## 4. DNS records at the registrar

First remove the registrar's parking page or web forwarding for the domain and for www. Then add these records (TTL 3600, or the registrar's default):

| Type  | Host / name                                | Value                          |
|-------|--------------------------------------------|--------------------------------|
| A     | @ (the domain itself)                      | 185.199.108.153                |
| A     | @                                          | 185.199.109.153                |
| A     | @                                          | 185.199.110.153                |
| A     | @                                          | 185.199.111.153                |
| AAAA  | @                                          | 2606:50c0:8000::153            |
| AAAA  | @                                          | 2606:50c0:8001::153            |
| AAAA  | @                                          | 2606:50c0:8002::153            |
| AAAA  | @                                          | 2606:50c0:8003::153            |
| CNAME | www                                        | gustavjiversen01.github.io.    |
| TXT   | _github-pages-challenge-gustavjiversen01   | the code from section 2        |

The www record points to the account's github.io address, without the repository name. GitHub then sends www.singulariteten.dk on to singulariteten.dk.

Check the records (dig is not installed on this machine; `sudo apt install dnsutils` adds it, or use the Google lookups):

```
dig +short singulariteten.dk A
dig +short singulariteten.dk AAAA
dig +short www.singulariteten.dk CNAME
curl -s "https://dns.google/resolve?name=singulariteten.dk&type=A"
```

DNS changes usually show within an hour but can take up to 24 hours.

## 5. HTTPS

1. Once the records resolve, GitHub requests a certificate by itself. This usually takes minutes, but can take up to a day.
2. When it is ready, enforce HTTPS:

   ```
   gh api -X PUT repos/Gustavjiversen01/singulariteten-site/pages -F https_enforced=true
   ```

   If this fails with a certificate error, the certificate is not ready yet: wait and run it again. If nothing happens for a day, remove the custom domain under the repository's **Settings > Pages**, then add it again.
3. Check all three addresses:

   ```
   curl -sI https://singulariteten.dk/ | head -1                       # HTTP/2 200
   curl -sI http://singulariteten.dk/ | grep -i '^location'            # https://singulariteten.dk/
   curl -sI https://www.singulariteten.dk/ | grep -i '^location'       # https://singulariteten.dk/
   ```

## 6. The redirect for indisingulariteten.dk

This section applies only if you register indisingulariteten.dk as well.

**Option A, simplest: the registrar's web forwarding.**

- Forward indisingulariteten.dk and www.indisingulariteten.dk to `https://singulariteten.dk`.
- Choose a permanent (301) redirect, keep the path, and no frame.
- Many registrar forwards work only over plain http. Test `https://indisingulariteten.dk` after setting it up; if that fails, use option B.

**Option B: a second small GitHub Pages site that redirects, with HTTPS.**

It is prepared in `redirect-indisingulariteten/`. Copy it out of this repository before making it a repository of its own:

```
cp -r ~/Documents/Investment/output/singulariteten-site/redirect-indisingulariteten \
      ~/Documents/Investment/output/indisingulariteten-redirect
cd ~/Documents/Investment/output/indisingulariteten-redirect
git init -b main && git add -A && git commit -m "Redirect indisingulariteten.dk to singulariteten.dk"
gh repo create indisingulariteten-redirect --public --source=. --remote=origin --push
gh api -X POST repos/Gustavjiversen01/indisingulariteten-redirect/pages \
  -f "source[branch]=main" -f "source[path]=/"
gh api -X PUT repos/Gustavjiversen01/indisingulariteten-redirect/pages -f cname=indisingulariteten.dk
```

- **DNS:** give indisingulariteten.dk the same eight A and AAAA records as in section 4, plus `www CNAME gustavjiversen01.github.io.`.
- **Verification:** verify the domain in GitHub as in section 2.
- **HTTPS:** enforce it as in section 5, using `indisingulariteten-redirect` in the repository path.
- **What it does:** every address on indisingulariteten.dk goes on to the same path on singulariteten.dk. This is done by a script, with a page refresh to the front page if scripts are off, and the canonical link points to singulariteten.dk.

## 7. After launch

- **The site:** open https://singulariteten.dk/ and https://singulariteten.dk/en/. Try:
  - DA / EN (the address follows the choice);
  - Listen and Download as PDF;
  - both PDF links;
  - the newsletter dialog;
  - a missing page such as https://singulariteten.dk/ikke-her for the 404 page.
- **Share previews:**
  - LinkedIn: paste the address into the Post Inspector (https://www.linkedin.com/post-inspector/). This also refreshes LinkedIn's copy.
  - X: shows the large card when a post contains the link.
  - The images are `docs/assets/share-da.png` and `share-en.png`.
- **Search (optional, needs a Google account):** in Google Search Console, add the domain (by a DNS TXT record) and submit https://singulariteten.dk/sitemap.xml.

## Updating the site

```
cd ~/Documents/Investment/output/final_k05/_build
python3 site.py && python3 launch.py && python3 launch.py check    # check saves screenshots to checks/
cd ~/Documents/Investment/output/singulariteten-site
git add -A && git commit -m "Describe the change" && git push
```

GitHub Pages republishes within a minute or two.

## Newsletter (MailerLite)

The signup sends to Gustav's MailerLite form. The settings are in `NEWSLETTER` in `final_k05/_build/site.py`:

- `form`: `https://assets.mailerlite.com/jsonp/2760610/forms/202962543441872796/subscribe`
- `field`: `fields[email]`
- `extra`: MailerLite's hidden fields, `ml-submit` and `anticsrf`

How it behaves:

- **Sending:** the address goes to MailerLite in the background. The dialog then says "Tak! Du er nu tilmeldt." / "Thanks! You’re now subscribed.", because double opt-in is off. If it is turned on in MailerLite, change `nl_thanks` in `site.py` to ask for the confirmation mail.
- **No answer back:** the page cannot see MailerLite's answer, so the form must be saved and active in MailerLite's dashboard; otherwise readers are thanked but not added.
- **Before announcing the newsletter,** send one real signup with your own address and confirm it.
- **To switch to Substack** (page mode) or change the form, edit `NEWSLETTER` and follow "Updating the site".

### The optional comment

Under the address there is an optional field: "Kommentar til os (ikke offentlig)" / "A note to us (not public)". The settings are in `COMMENTS` in `site.py`.

- **No comment:** nothing changes. The signup sends exactly what it sent before.
- **With a comment:**
  - MailerLite gets it as `fields[comment]` on the subscriber, on one line. It is stored only if the subscriber field `comment` exists in MailerLite.
  - Once the signup has gone, the comment is also emailed to kommentar@singulariteten.dk through FormSubmit (`https://formsubmit.co/ajax/kommentar@singulariteten.dk`). The email includes the address, the page language, the time and the page.
- **Limits:** a comment is at most 1,000 characters, because MailerLite refuses the whole signup if any field is longer than 1,024.
- **Robots:** an invisible honeypot field means a robot's comment is sent nowhere.
- **If the comment service fails,** the signup is not affected.
- **FormSubmit activation:** its first submission sends an activation email to kommentar@. That link must be clicked once, and comments sent before it are delivered after activation.
- **To switch to another service,** for example Web3Forms, change `url` and `extra` in `COMMENTS`.

## Read-aloud and PDF buttons

At the head of the paper sit "Lyt" / "Listen" and "Download som PDF" / "Download as PDF".

- **Listen** opens a player docked at the foot of the window. It has play and pause, a seek line with
  a tick at each chapter, a chapter menu, the time left, speed (1×, 1,25×, 1,5×) and a close button.
- **The audio files** are `docs/audio/Ind-i-singulariteten_DA.mp3` and `Into-the-Singularity_EN.mp3`.
  They are fetched only when Listen is pressed (`preload="none"`), so the page itself stays light.
- **How the audio is made:** the steps and the model are in `~/Documents/Investment/output/tts/README.md`.
  The audio is not rebuilt automatically, so after a text change make the audio again before
  rebuilding the site.
- **Download as PDF** downloads the current language's PDF, the same file as the "PDF" link in the colophon.

## Privacy and analytics

- **No cookie banner is needed.** The site sets no cookies, stores nothing in the browser (no localStorage), and loads nothing from other sites: fonts, figures and scripts are all inside the pages. The local check confirms all of this (`checks/report.json`).
- **Analytics are off.** `ANALYTICS = False` in `final_k05/_build/launch.py` is the one-line switch. True adds Plausible's cookieless page counts. Plausible is EU-hosted and needs a paid account at plausible.io with singulariteten.dk added. No cookie banner is needed with it either.

## Social images (assets/social/)

| File                                   | Where it goes                                           |
|----------------------------------------|---------------------------------------------------------|
| linkedin-page-logo-400x400.png         | LinkedIn page: Edit page > Logo                         |
| linkedin-page-cover-1128x191.png       | LinkedIn page: cover image                              |
| linkedin-profile-banner-1584x396.png   | a personal LinkedIn profile: background photo           |
| x-avatar-400x400.png                   | X: profile photo (X crops it to a circle)               |
| x-header-1500x500.png                  | X: header                                               |
| substack-logo-512x512.png              | Substack: publication logo                              |
| substack-email-banner-1100x220.png     | Substack: email banner                                  |

On LinkedIn and X the profile photo covers the lower left of the header, so the title sits on the right there. The avatars keep the dots inside a circle, so a round crop cuts nothing off.
