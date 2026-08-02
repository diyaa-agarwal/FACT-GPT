# claimed_contribution.md
## Exact Project Claims

---

## What We Reproduced

**1. Textual entailment as the core fact-checking mechanism (from Choi & Ferrara, 2024)**
We reproduced the three-class textual entailment formulation (ENTAILMENT → SUPPORTS, CONTRADICTION → REFUTES, NEUTRAL → NEUTRAL) using RoBERTa-large-MNLI. We did not reproduce their fine-tuning pipeline — we use the publicly available RoBERTa-large-MNLI checkpoint directly without any additional training on synthetic data.

**2. Check-worthiness scoring inspired by CLEF-2023 CheckThat! Task 1B (Alam et al., 2023)**
We implemented a heuristic scoring function that operationalises the same two annotation questions used by CheckThat!: "does it contain a verifiable factual claim?" and "is it harmful?" We did not reproduce the XLM-RoBERTa or GPT-3 fine-tuned classifiers that top CheckThat! teams used; our scorer is a rule-based heuristic, significantly weaker than those systems.

**3. NLLB-200 translation for Devanagari and Bengali**
Standard use of the facebook/nllb-200-distilled-1.3B model for Hindi and Bengali to English translation. No fine-tuning performed; translation quality is as-is from the pretrained checkpoint.

**4. GoEmotions multi-label emotion classification (first notebook only)**
In FACT_GPT_complete__3_.ipynb we trained RoBERTa-base on the GoEmotions dataset from scratch for 3 epochs. This was subsequently dropped in the deployed v2 system in favour of the pretrained SamLowe/roberta-base-go_emotions HuggingFace checkpoint, used directly without any fine-tuning.

---

## What We Modified

**1. Live evidence retrieval replacing static corpus lookup**
Choi & Ferrara match claims against a fixed corpus of 1,225 debunked claims. We replaced this entirely with real-time DuckDuckGo web search and Wikipedia retrieval at query time. This is the most significant architectural departure from the base paper, motivated by the need for temporal currency (a static COVID-19 corpus is useless for general Indian WhatsApp claims).

**2. Weighted multi-snippet aggregation to address CONTRADICTION weakness**
Choi & Ferrara report F1_Con ≤ 0.46 across all models on a single entailment prediction per claim. We aggregate weighted votes (w_i = similarity_score_i × NLI_confidence_i × source_trust_i) across up to 5 independent evidence snippets. This partially mitigates the single-prediction weakness but does not fully solve it — our own FALSE verdict precision remains lower than TRUE verdict precision.

**3. Snippet-based RAG replacing full-page HTML fetching**
An early version of our system fetched the full HTML of each search result URL (~6 requests, ~41 seconds total). We switched to using DuckDuckGo result snippets (~200 characters each) directly as evidence, with full-page fetching only as a last resort. This reduced the RAG stage from 41.3 seconds to 1.8 seconds with approximately 2 percentage points of accuracy trade-off. This is an engineering optimisation, not a modelling contribution.

**4. Parallelised Wikipedia and DuckDuckGo retrieval**
Wikipedia and DuckDuckGo search now run as concurrent threads (Python ThreadPoolExecutor, 2 workers, 12-second timeout each). This reduced combined retrieval time from approximately 18 seconds (sequential) to approximately 8.6 seconds. Pure engineering change; no effect on verdict accuracy.

**5. Async Twilio webhook architecture**
Twilio's webhook imposes a hard 15-second HTTP timeout. Our pipeline takes 15–40 seconds. We solved this with a two-phase design: Flask returns HTTP 200 with an acknowledgement message immediately, then a daemon background thread runs the full pipeline and delivers results via the Twilio REST API. This is a deployment engineering contribution required to make the system functional; it is not a machine learning contribution.

**6. Hinglish script detection via lexical signals**
We extended Unicode character-count script detection with a 40-token vocabulary of Roman-Hindi signal words (hai, nahi, mein, kya, bhi, woh, kiya, raha, gaya, hua, ne, ke, ka, ki, tha, thi, etc.) to identify Hinglish text in Latin script. One token match classifies a message as Hinglish. This is a simple addition not present in any base paper.

**7. Indian political alias resolution**
An 11-entry dictionary maps common aliases to canonical names: Pappu → Rahul Gandhi; Modiji/NaMo/Feku → Narendra Modi; Didi → Mamata Banerjee; AK49/Kejru → Arvind Kejriwal; Behenji → Mayawati; MMS → Manmohan Singh. Applied via word-boundary regex before claim extraction.

**8. WhatsApp-specific preprocessing**
Regex removal of forwarded-message banners in English and Devanagari Unicode variants, URL stripping, excessive punctuation normalisation, and emoji compression. Tailored to Indian WhatsApp message patterns; not present in any base paper.

**9. Corpus disabled for verdict decisions**
We initially planned a self-improving corpus that saved high-confidence verdicts for future lookups. After discovering that auto-learned entries introduced systematic false positives and degraded accuracy, we disabled corpus-based verdicts entirely (auto-save threshold set to > 1.0). The corpus is now display-only. This was a corrective finding, not a planned contribution.

---

## What Did Not Work

**1. Hinglish translation via NLLB**
Routing Hinglish (romanised Hindi in Latin script) through NLLB with source code hin_Deva produces unreliable results. NLLB was trained on native-script Devanagari Hindi, not romanised Latin-script Hindi. For simple factual claims such as "Delhi Pakistan ka capital hai", NLLB sometimes produces garbled, incomplete, or identical-to-input output. On CPU (when GPU is unavailable), NLLB takes 8–12 seconds for Hinglish inputs even when the result is poor. When the translation fails, the pipeline falls back to the raw Hinglish text, which English NLI cannot reliably process. We did not find a satisfactory fix for this within the project timeline. The planned fix (routing Hinglish to Gemini which handles romanised Hindi natively) was not implemented in the deployed version.

**2. Complex Devanagari Hindi translation**
For simple Hindi claims, NLLB translation is functional. For longer, multi-clause Hindi sentences, NLLB-distilled-1.3B produces noticeably degraded English output. NLI then operates on awkward translations and misclassifies the stance. We do not claim reliable Hindi fact-checking for complex sentences.

**3. Bengali evidence retrieval**
Bengali claims translate adequately to English via NLLB. However, DuckDuckGo and Wikipedia return predominantly English-language results. Translated Bengali claims often match topically unrelated English articles, yielding irrelevant evidence and NOT VERIFIED verdicts even for clearly true or false claims. Our accuracy on Bengali in live tests is 1 out of 3 messages correct (33%). This is not reliable.

**4. GoEmotions fine-tuning from scratch**
In the initial prototype, we fine-tuned RoBERTa-base on GoEmotions for 3 epochs (micro-F1 ≈ 0.58). The pretrained SamLowe/roberta-base-go_emotions checkpoint performs similarly or better with no training cost. We abandoned our fine-tuned version.

**5. Auto-learning corpus**
A self-improving corpus that saved verified claims for future retrieval was planned and partially implemented. In practice, incorrect early verdicts contaminated the corpus and degraded subsequent lookups. The feature was disabled.

**6. RSS feed-based retrieval (first prototype)**
The initial v1 system used 8 curated RSS feeds (PIB India, WHO, NDTV, etc.) and the GNews API. RSS feeds were stale for breaking news, and GNews had a 100-request/day free-tier limit. This approach was replaced by DuckDuckGo + Wikipedia.

**7. ngrok persistence**
The free ngrok tier terminates sessions after 2 hours, requiring manual Twilio webhook URL reconfiguration. This makes the bot non-viable for extended deployment without a paid plan or persistent server.

---

## What We Believe Is Our Contribution

We make no claims of novel machine learning methodology. Our contributions are engineering and systems contributions applied to an underserved context (Indian multilingual WhatsApp). We state these precisely:

**1. A working deployed WhatsApp fact-checking bot for Indian languages**
We deployed a live chatbot via Twilio Sandbox, Flask, and ngrok on Google Colab, and demonstrated correct verdicts on live WhatsApp messages (two screenshots included in the report). To our knowledge this is the first BITS Pilani CS F425 project to demonstrate end-to-end live fact-checking via WhatsApp with multilingual support. The pipeline handles English reliably (83% accuracy on our test set) and Hinglish partially (75%, with known latency issues).

**2. Async two-phase webhook architecture for Twilio compatibility**
The design of returning an immediate HTTP 200 ACK and delivering results via Twilio REST API from a background thread is a non-trivial systems engineering contribution. It makes an ML pipeline with 15–40 second latency compatible with Twilio's hard 15-second HTTP constraint. This pattern would be reusable for any NLP chatbot deployed on Twilio.

**3. Latency optimisation analysis for a deployed NLP pipeline**
We identify and quantify three independent bottlenecks — HTML fetching (−39.5 s), sequential retrieval (−9.8 s), and beam search width (−4.1 s) — reducing total latency from 84 seconds to 25 seconds (3.3× speedup). This kind of end-to-end profiling and targeted engineering is underreported in academic NLP work and has direct practical value.

**4. Hinglish-aware script detection**
A lightweight but practical extension of Unicode-range script detection to handle romanised Hindi, a code-mixing variety completely absent from Choi & Ferrara and all CheckThat! editions. The 40-token lexical signal approach is simple, interpretable, and sufficient for routing purposes.

**5. Indian-context preprocessing and alias resolution**
The combination of WhatsApp artefact removal, emoji compression, and Indian political alias resolution constitutes a practical preprocessing contribution for Indian social media that extends the English-centric base papers to a new domain.


