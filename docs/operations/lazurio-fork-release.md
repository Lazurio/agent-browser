# Lazurio distribuce agent-browser

`Lazurio/agent-browser` je veřejný distribuční fork
[vercel-labs/agent-browser](https://github.com/vercel-labs/agent-browser)
(Apache-2.0). Lazurio ho instaluje na Remote Environmenty jako prohlížeč
Environmentu: jeden Chrome s jedním profilem na virtuální obrazovce, okno pro
každé vlákno agenta a pohled, ve kterém člověk sleduje a převezme ovládání
(root rozhodnutí 0191, LazurioPlatform F38, plán DEV-6646). Fork nese jen
opravy, které Environmenty potřebují dřív, než je vydá upstream; každou
zároveň nabízí upstreamu.

## Model: rolling distribution patch-stack

Stejně jako `Lazurio/t3code` a `Lazurio/OpenMausBot` (HumanAndMachine-ai
`AGENTS.md` §6, `ARCHITECTURE.md` §7, skill `upstream-rebase-fork`):

- `main` je čitelný snapshot právě publikované distribuce: exact upstream
  release tag a nad ním pouze aktuální, významově oddělené Lazurio commity.
- Historickou a rollback autoritou jsou chráněné immutable release tagy
  `vX.Y.Z-lazurio.N`, ne `main`.
- `main` smí nahradit jen Organization Admin, exact
  `--force-with-lease=refs/heads/main:<expected-old-main-sha>`, po zeleném
  candidate CI, approvalu na přesném candidate HEADu a až když je starý
  `main` zachycen release tagem. Jedinou úlevou byl jednorázový bootstrap:
  první nahrazení `main` z upstream commitu
  `6d3e22c673a44271d0c213c2fef722e0aeba627d` (upstream `main` při založení
  forku), ověřeného těsně před zápisem jako předka čerstvého upstream `main`.

## Aktuální báze a patche

Báze: upstream `v0.37.0` (`471ab3852b47b98847f1d9c855c272bb62d0d50b`), verze,
kterou Machines nasazovaly před forkem.

| Commit | Co opravuje | Proč ho nese Lazurio | Kdy zmizí |
| --- | --- | --- | --- |
| `fix(dashboard): use VK_OEM codes for punctuation in viewport keys` (cherry-pick vercel-labs/agent-browser#1382) | Pohled posílal každý znak s ASCII kódem jako Windows virtual-key kód, takže `.` dorazila jako Delete, `-` jako Insert, `'` jako šipka vpravo (vercel-labs/agent-browser#1380) | Člověk, který v pohledu převezme přihlášení, nenapsal e-mail s tečkou | Až upstream #1382 (nebo jinou opravu #1380) vydá a distribuce přejde na takové vydání |
| `fix(dashboard): serve the SPA shell as HTML when the request carries a query` (cherry-pick vercel-labs/agent-browser#2046) | Dashboard servíroval `/?port=…` jako `application/octet-stream` | Odkaz na okno vlákna (`/?port=<stream>`) se v prohlížeči stáhl místo otevřel; LazurioPlatform to obchází parametrem `view=.html` | Až upstream #2046 vydá; obejití v Platformě pak může odejít |
| `chore(lazurio): allow clippy::double_must_use on BrowserBackend, as upstream main does` | Dva řádky z upstream `main` (vercel-labs/agent-browser@63df443): současný stabilní clippy odmítá `#[must_use]`, které `async_trait` přidává k metodám `BrowserBackend` | Upstreamové CI (clippy s `-D warnings` na nejnovějším stabilním Rustu) by jinak na bázi v0.37.0 neprošlo | Až distribuce přejde na upstream vydání, které ty řádky obsahuje |
| `chore(lazurio): distribution version …` | Verze `X.Y.Z-lazurio.N` všude, kde ji upstream drží; upstreamový npm release běží jen ve `vercel-labs/agent-browser` | Distribuce se vydává tady, ne na npm | Zůstává |
| `chore(lazurio): release workflow and distribution contract` | `.github/workflows/lazurio-release.yml` a tento dokument | Vydání s atestací, které Machines pinují | Zůstává |

Další kandidát: vercel-labs/agent-browser#1618 (latence vstupu, klávesové
zkratky, schránka). Do distribuce přijde až po vlastním review.

## Verze

`X.Y.Z-lazurio.N`: `X.Y.Z` je upstream release, na kterém `main` stojí, `N`
začíná na 1 a roste s každým vydáním nad stejnou bází; nová báze začíná znovu
od `.1`. `agent-browser --version` vypíše `agent-browser X.Y.Z-lazurio.N`.

Pozor na SemVer: `0.37.0-lazurio.1` je nižší než `0.37.0`. Machines proto
binárku upstreamu, kterou dřív samy nainstalovaly, poznají podle digestu
(`superseded_machines_binaries`) a nahradí ji; jiná instalace s vyšší verzí
zůstává.

## Vydání

Brány, které drží repozitář (nastavuje je Organization Admin; schvalovatel je
před schválením přečte zpět):

- prostředí `lazurio-agent-browser-release`: povinný reviewer `immakermatty`,
  `prevent_self_review: true` (vydání spouští někdo jiný, než kdo ho schvaluje)
  a nasazení jen z větve `main`:
  `gh api repos/Lazurio/agent-browser/environments/lazurio-agent-browser-release`;
- immutable releases zapnuté:
  `gh api repos/Lazurio/agent-browser/immutable-releases` → `"enabled": true`
  (token workflow nastavení číst neumí, proto ho workflow dokazuje až na
  publikovaném vydání);
- ruleset „Protect Lazurio release tags“: `refs/tags/v*-lazurio.*` nejde
  přepsat ani smazat.

1. `main` je candidate po review a zeleném CI.
2. Spusť workflow **Lazurio agent-browser Release** (`workflow_dispatch`) se
   vstupy `version` (shodná s `package.json`), `source_sha` (přesná špička
   `main`), `upstream_tag` a `upstream_sha` (přesný commit upstream tagu).
3. Workflow odmítne spuštění, které neběží z `refs/heads/main` přesně na
   `source_sha`, ověří, že zdroj je špička `main`, leží nad upstream tagem jen
   lineárními Lazurio commity a nese verzi, sestaví `agent-browser-linux-x64`
   (glibc 2.28) a `agent-browser-darwin-arm64`, ověří, že binárka hlásí verzi
   vydání, a čeká na schválení prostředí `lazurio-agent-browser-release`.
4. Po schválení znovu ověří spuštění a špičku `main`, zapíše `SHA256SUMS`,
   atestuje obě binárky, publikuje release `vX.Y.Z-lazurio.N` s tagem na
   `source_sha` a ověří, že je immutable a tag míří na `source_sha`; jinak
   běh selže.

Ověření binárky:

```sh
gh attestation verify agent-browser-linux-x64 \
  --repo Lazurio/agent-browser \
  --signer-workflow Lazurio/agent-browser/.github/workflows/lazurio-release.yml \
  --source-digest <source_sha>
```

Machines pak pinují `agent-browser-linux-x64` vydání digestem SHA-256 a
velikostí; pohyblivý `main` nepinuje nic.

## Refresh na nové upstream vydání

Podle skillu `upstream-rebase-fork`, režim rolling distribution patch-stack:

1. Candidate začíná na exact upstream release tagu, nikdy na pohyblivém
   upstream `main`.
2. Každý patch se znovu klasifikuje `remove` (upstream ho vydal), `retain`
   nebo `migrate`; starý `main` se do candidate nemerguje.
3. Verze `X.Y.Z-lazurio.1` nové báze, candidate PR s approvalem na přesném
   HEADu a zeleným CI.
4. Starý `main` musí být zachycen release tagem; pak exact-lease nahrazení
   `main` a samostatné vydání.
