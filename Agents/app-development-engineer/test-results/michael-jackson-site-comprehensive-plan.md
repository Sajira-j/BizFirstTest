# Michael Jackson Site — Plan for a Comprehensive, Professional Rebuild

The current site (AppID 1064) is a proof-of-concept: 4 thin pages, one real image, enough content
to prove the MCP build pipeline works end to end. This plan is for turning it into something
genuinely presentable — real depth, real design, built the same way (MCP tools), reviewed the same
way (live design-review pass in the Player), but with far more substance.

## What App Studio can and can't give us — read before planning content

App Studio's widget palette is real but simple: `content` (rich text/markdown), `image`, `video`,
`audio`, `pdf`, `page-navigation`, plus 4 gallery types that need real uploaded, tenant-flagged
public assets (no MCP tool can create those yet — not usable for this site without that gap being
closed first). There's no dedicated timeline/accordion/card-grid widget the way Atlas Forms has —
richness here comes from **more pages, better-organized sections, and real design polish (styling,
imagery, typography)**, not exotic widget types. `video` widgets can embed a real YouTube URL
(linking to existing copyrighted footage, not hosting a copy of it — the same practice any tribute
site uses) which is the one way to add real video content without a licensing problem.

## Proposed site structure — 7 pages instead of 4

| Page | Content plan |
|---|---|
| **Home** | Hero image, full name/dates, one-paragraph overview, pull-quote, nav to every other page |
| **Early Life & The Jackson 5** | Born 1958 Gary, Indiana; family group act from age 6; Motown years; first solo chart success ("Got to Be There," "Ben") |
| **Solo Career & Thriller Era** | *Off the Wall* (1979), *Thriller* (1982) — still the best-selling album of all time, 7 Hot 100 top-10 singles, the Moonwalk's TV debut (Motown 25, 1983), the short-film-style videos that changed MTV |
| **Bad, Dangerous & HIStory** | *Bad* (1987, first album with 5 U.S. #1s), *Dangerous* (1991), *HIStory* (1995), stadium tours, the Super Bowl XXVII halftime show |
| **Awards & Records** | 13 competitive Grammys + Legend Award, Rock & Roll Hall of Fame (solo 2001 + Jackson 5 1997), Songwriters Hall of Fame, Guinness Records ("Most Successful Entertainer of All Time," 1984) |
| **Humanitarian Work & Legacy** | "We Are the World" (1985 co-write), Heal the World Foundation, influence on subsequent pop/R&B/dance artists, continued sales/streaming relevance |
| **Gallery / Media** | The verified-license images collected for this project, plus 1-2 embedded YouTube videos (real performances) via the `video` widget |

Each content page: a `header`-style opening via `content` widget, 2-4 real paragraphs (factually
checked, not fabricated specifics), a relevant image where one exists, and a closing pull-quote —
matching the pacing that already worked well on the current Home page, just repeated with real
depth instead of one page's worth of content spread thin.

## The real constraint: images

Michael Jackson is a heavily-copyrighted subject — most professional photography of him is **not**
freely licensed, and this project will not use an image without a verified license. Tonight's site
uses exactly one real, confirmed-public-domain photo (a 1984 White House Photo Office image,
license verified on its actual Wikimedia Commons page, not assumed). That's a real, structural
limit on this plan, not a shortcut — Wikimedia Commons' Michael Jackson category has a small number
of similarly-verifiable images (mostly official/government/press-office photos, or performance
photos a photographer specifically released under a free license) and it's worth methodically
working through that category rather than guessing.

**Where your help matters, as you offered**: if you already have (or want to source) specific
images with verified usage rights — a personal collection, stock photography you have a license
for, or specific Wikimedia Commons files you've already checked — supplying the exact URLs (and
confirming the license) would let this plan use real, varied imagery per page instead of one
photo stretched across seven. Concretely useful: one image per page (7 total), ideally spanning
different eras (Jackson 5 childhood photo, 1980s Thriller-era, 1990s, a stage/performance shot).
If specific images aren't available, the plan still works with fewer, reused thoughtfully (e.g. the
one verified 1984 photo on Home + the Awards page, no invented "stand-in" photos in between).

## Design approach

- **Consistent visual system across all 7 pages** — one color palette, one heading/body type scale,
  applied via `update_widget_placement`'s `styleConfiguration`/`widgetStyle` on every content
  widget, not styled ad hoc per page (the current site's one visible styling pass — padding/max-width
  on the Home page's content block — should become the template every other page's content widget
  copies, not a one-off).
- **A real color/type direction, not default**: something that reads considered for the subject —
  e.g. a dark ground (near-black) with a single accent (gold or red, both period-appropriate to the
  Thriller/Bad album art era) for headers/pull-quotes, generous line-height and paragraph spacing
  for the biography text blocks, consistent hero-image treatment (max-width, centered, consistent
  aspect ratio) across every page that has one.
- **Real page-navigation widget wiring** — all 7 pages linked from one shared nav placed identically
  on every page (reusing the same widget/section pattern, not rebuilt per page).

## Execution plan

1. Confirm the design direction above (or adjust) before building — this is the one part worth a
   quick yes/no rather than discovering a wrong direction after 7 pages are built.
2. If you're supplying images, get URLs + confirmed licenses first — content and image placement
   are easier to build together than content-first-then-retrofit-images.
3. Build via the same MCP path as tonight (`create_page` ×3 more, `create_section` per page,
   `create_widget`/`update_widget_placement` for content + images + nav), in one pass per page
   rather than one MCP call at a time, to keep this efficient.
4. Real design-review pass in the Player after all 7 pages exist — screenshot every page, check
   consistency (not just "does it render," which tonight's testing already covers), fix anything
   that reads as unpolished, re-screenshot to confirm.
5. Update `app-studio-mcp-coverage.html`'s "what this proves" framing once done — this becomes the
   real, professional-grade demonstration of the whole pipeline, not just the proof-of-concept.

## Open questions for you

1. Does the 7-page structure above look right, or do you want a different cut (e.g. split
   Awards/Legacy into two pages, or add a dedicated Discography page listing every studio album)?
2. Any specific images you can supply (with confirmed licenses), or should this proceed with the
   single verified photo reused thoughtfully across pages?
3. Any preference on the color/type direction, or is "considered, period-appropriate, one accent
   color" enough direction to proceed?
