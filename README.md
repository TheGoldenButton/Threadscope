# Concept Field

A single-file (`xrayvision.html`) tool that turns your text into a navigable 3D map of concepts. Words that tend to appear near each other in your documents are drawn close together and linked. You explore the map by asking questions, clicking words, and reading the source passages behind each connection. It runs in the browser, on desktop, in VR, or in mixed reality on Quest. There is no build step and no server.

## Quick start

1. Open `xrayvision.html` in a modern browser (needs WebGL; WebGPU only if you use local LLMs).
2. Click **+** and add text: files (txt, md, csv, json, html, PDF, docx), Wikipedia pages, pasted text, a URL, or the built-in demo (oceans and climate).
3. Wait for "Building field…". The concept map appears.
4. Type a question (e.g. *how does heat affect the ocean?*) or click words.

Everything is stored locally (IndexedDB and localStorage). Your last text set reloads automatically.

## How the system works

### 1. Ingest: text into tokens
All heavy work runs in a Web Worker (the "engine") so the UI stays responsive.

- **Tokenizing.** Text is lowercased and split into words, numbers, and punctuation. Punctuation and paragraph breaks are kept as *boundary markers* rather than discarded.
- **Filtering.** Stop words, structural words (numbers, time words, prepositions), contractions and junk are dropped from the drawn field, but they are not lost: every word you type in a query is still searched.
- **Plural folding.** *waters* and *water* are merged into one concept.
- **Phrase merging.** Word pairs that co-occur far more often than chance (pointwise mutual information) are fused into multi-word concepts, up to 5 words, over several passes.
- **Idea boundaries.** Sentences are compared against the running profile of the topic so far (IDF-weighted word vectors). A sentence that is less cohesive with that profile than adjacent sentences typically are in this text starts a new topic. Scans never cross these boundaries, so association does not leak between topics, on short texts as well as long ones. Nothing about window size is fixed; the threshold comes from the text's own cohesion.

### 2. Association: which words belong together
For each word, the engine scans forward and backward from every occurrence and accumulates weight for the words nearby.

- Weight decays with distance. Crossing a sentence or paragraph break reduces it further; short paragraphs let association pass through more easily.
- Scores are weighted by **IDF-like rarity** (inverse window frequency), damped for generic words, and compared against what chance would predict from each word's overall frequency. Words that co-occur *more than expected* score high.
- **Rare-word floor.** A word appearing in at most 3 windows, but occurring in the same topic segment as the seed (see Idea boundaries), gets a rarity lift: its score becomes `cut + (1 - cut) * score`, where `cut` is the Query pruning cut-off, so every rare in-topic word clears the display cut-off but keeps its own proximity ordering (no flat ties). Words in other topic segments are never reached, so they get no lift. This is the `rareW` check in `leapScan`.
- Direction is tracked, so each link knows whether A tends to precede B.

### 3. The field: layout
- The top ~3,000 concepts by frequency × rarity become candidates. Each keeps its 6 strongest links.
- Communities are found by label propagation, then a 3D force-directed layout (Barnes-Hut repulsion for large graphs, spring attraction along links, extra cohesion inside communities) positions the nodes. A final pass separates overlapping nodes.
- Node brightness and size reflect frequency. Zooming out collapses neighbouring words into the most dominant one; zooming in expands them.
- **The field learns.** Words you query or select together are nudged slightly closer, and the arrangement is saved per text.

### 4. Querying: concept exploration
When you ask a question, the engine does not do keyword lookup. It:

1. **Seeds** from your words, including inflections and close spelling variants.
2. **Scans** each seed's neighbourhood across the corpus (first-order associations).
3. **Re-seeds** from the best of those and scans again (second-order, "emerged" words that connect your terms indirectly).
4. **Interferes** the per-term score maps. Words that associate with *all* your terms, not just one, rise; this is multiplied, not summed, so it acts like an AND with soft edges. Coverage, mutual-pair strength, and a diversity step remove redundant near-duplicates.
5. **Prunes** by the *Query pruning* cut-off in Settings (relative to the best connection).
6. Optionally refines with musical-consonance and precedence heuristics (which words tend to come before or after your terms).

Result: seeds glow amber, emerged words light up according to resonance, and the links between them are drawn bright. A ranked list appears in the info box.

**Selecting concepts.** Click words to build a selection; the system then queries them as if you had asked them together. Click a **link** to see where its two words co-occur.

### 5. Source passages
The **Passages** panel shows the actual sentences behind a selection, query, or link: matched words highlighted, document name and position shown, ranked by relevance. How many appear adapts to how strongly the words resonate. Copy exports them with citations.

### 6. Play
With a query or selection active, **Play** flies through the words in resonance order, strongest first, chiming each one on a pentatonic scale tied to its frequency and connectivity. Speed cycles from 0.25× to 4×.

### 7. Multiple documents
Two layouts, switchable from the file count button:

- **Unified.** All texts form one field.
- **Separated orbs.** Each document gets its own sphere, in its own colour. A query draws **cross-bridges** between orbs: gold/white lines for the same word in two files, and labelled links for different words that serve the same role. Words that are strong in one file but absent from the others appear as **red divergence markers**, the blind spots. Drag an orb's label to move it.

### 8. Optional LLM assistant
Enable in Settings. Providers: local WebGPU models (no key), Claude, ChatGPT, Gemini, Muse, or OpenRouter (API keys stay in your browser and go only to the provider you pick).

The assistant receives JSON: your query, the relevant words (above the relevance threshold), and the top passages. It answers from those and cites them like `[P2]`. Follow-ups reuse the same context. **Copy** gives you the full prompt to paste into any other model. **Auto** runs it on every query. A token-saver cut-off sends only high-scoring passages.

### 9. VR / mixed reality
Enter VR or mixed reality (Quest 3 passthrough) to walk around the field. Pinch to select, pinch-drag to move, two hands to resize and rotate, and the field remembers where you placed it. Floating panels provide a keyboard, results box, passages, settings, Wikipedia search, and file manager. Open your palm for voice input, which runs Whisper in-browser. You can say "Wikipedia: topic" to pull a page in.

## Voice and Wikipedia commands
Typed, dictated, or spoken: `Wikipedia: <topic>` fetches that page. Add *as its own sphere* or *into the unified space* to choose the layout.

## Settings that matter

| Setting | Effect |
|---|---|
| Query pruning | How strongly an emerged word must connect your query terms (0 to 1). Higher is stricter. |
| Rainbow lines | Colour lines by word frequency (rare = violet/blue, common = orange/red) vs. link strength. |
| Relevance threshold | Minimum score for a word to be sent to the LLM. |
| Passage cut-off | Token saver: only passages at or above this relevance go to the LLM. |
| Font size | Scales labels and panels, including in VR. |

## Tips
- **Short texts** give sparse structure. Use a few thousand words or more for rich results; the demo is deliberately small.
- Use specific terms; very common words are searched but contribute little signal.
- Select several words and read the passages: that is the quickest way to see *why* two concepts are linked.
- PDFs must contain real text. Scanned PDFs need OCR first.
- Some sites block direct URL fetches (CORS); the optional proxy retry sends the URL to a public proxy.

## Files
- `xrayvision.html`: the entire application (UI, rendering, worker engine).
- Persisted data: IndexedDB database `conceptfield` (stores `corpora`, `history`, `fields`), plus settings in localStorage keys prefixed `cf.`.

## Change log
- **Rare-word floor now applies to short texts.** The `NWIN>=3` condition was removed from the `rareW` check in `leapScan`.
