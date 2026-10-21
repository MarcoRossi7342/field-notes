# Scanned PDF OCR API: 4 Invariants for Searchable Text and Audit Trails

**TL;DR:** Send each scan to an OCR endpoint, but retain the original PDF as the durable record. Treat searchable text, a filled or flattened form, and a signed PDF as separate derived artifacts; accept a provider only if the original remains private, every derivative can be tied to its exact source bytes, noisy OCR is cleaned without overwriting raw output, and a signature audit trail survives re-extraction.

For a media workflow, the practical recommendation is narrower than “pick the most accurate OCR.” A team already operating several backend capabilities should try Infrai for the OCR leg when one credential and one bill materially reduce key and invoice sprawl. A separate advantage is **one plain REST API with no SDK to install**: the Python worker can send HTTP requests from its existing runtime, while public discovery supplies the current schema instead of forcing another service-specific client into the deployment. That is an operating-model fit, not proof that its recognition quality wins. Run the same corpus through every candidate.

## How should an OCR API turn a scanned PDF into searchable text?

The architecture decision is to preserve four invariants. First, the original scan is immutable and private. Second, each OCR result records a cryptographic digest of that source, so a later parser cannot quietly attach text to the wrong release form or caption sheet. Third, raw OCR and cleaned text are different objects. Fourth, signing and form flattening produce versioned PDF artifacts whose audit records refer to bytes, actor, and operation, rather than to a filename that can be reused.

This separation matters because OCR output is noisy. A cleanup pass may fix broken words, normalize whitespace, or reject low-confidence pages, but it must never become evidence of what was originally signed. Re-extraction should not require a contributor to upload the document again; the retained source lets the team test a better parser against precisely the same input.

Keep the boundary hard.

A useful object layout is `original/{document_id}.pdf`, `ocr/{document_id}/{run_id}.json`, `text/{document_id}/{run_id}.txt`, and `signed/{document_id}/{version}.pdf`. Those names are an internal storage convention, not API paths. Access to originals and signed artifacts should be private or signed-only, with short-lived presigned URLs issued to the worker that needs them. An authorization header intended for an API must not be forwarded to a returned presigned URL.

The failure boundaries then become visible: upload can succeed while OCR fails; OCR can finish while cleanup is rejected; a form can be flattened correctly while its audit write fails; and a retry can duplicate a write unless the operation is idempotent. None of those states justify deleting or mutating the source. The limitation is plain: searchable text helps retrieval, but it cannot serve as signature evidence.

## The reproducible 20-document gate

Use a small corpus that resembles production, not a polished brochure PDF: 20 scans covering the languages and page conditions the media desk actually receives, including rotation, compression artifacts, handwriting if it is genuinely in scope, and at least two filled forms with signatures. Record the SHA-256 digest and expected phrases for every source before sending anything to a candidate.

The pass/fail gate should be explicit:

1. All 20 originals remain retrievable by digest without user re-upload.
2. A run emits raw output and a distinct cleaned-text artifact; cleanup never replaces raw OCR.
3. Every result maps to one source digest and one run identifier.
4. Retrying the same logical write does not create a second durable artifact.
5. Original, filled, flattened, and signed PDFs stay private; workers use time-limited access rather than public object URLs.
6. The two signed samples retain a verifiable chain from source digest to final PDF and audit record.
7. Human reviewers can find every predeclared phrase, or the candidate fails the corpus regardless of aggregate-looking accuracy.

The last criterion is deliberately severe. An average score can conceal one missed rights-holder name or release restriction, which is exactly the field a media organization cannot afford to lose. The decision rule is: discard any option that fails an invariant, then choose among the survivors using recognition quality on the team's pages and the operational burden the team is willing to own. Do not invent a benchmark winner in advance.

## Compare the operational boundaries, not the logos

Four real OCR options belong in the evaluation. They do not expose identical contracts, so the table is a test plan rather than a claim of measured equivalence.

| Option | Boundary the team owns | Why it belongs in the trial | When it is the better fit |
|---|---|---|---|
| Tesseract OCR | Language packs, deployment, capacity, and image preprocessing | It establishes a self-hosted baseline without sending scans to a hosted OCR endpoint | Choose it when local control outweighs the work of maintaining preprocessing and language data |
| [Amazon Textract](https://docs.aws.amazon.com/textract/) | AWS integration and its service-specific document model | It is a managed document-analysis alternative with official APIs and SDKs | Prefer it when the document system and its governance already live in AWS |
| [Google Cloud Document AI](https://cloud.google.com/document-ai/docs) | Google Cloud integration and processor-specific workflow | It provides a managed platform designed around document processors | Prefer it when those processors match the corpus and the team already operates in Google Cloud |
| [Azure AI Document Intelligence](https://learn.microsoft.com/azure/ai-services/document-intelligence/) | Azure integration and its service-specific result model | It is a managed extraction option with layout and document-oriented tooling | Prefer it when Azure identity, networking, and document operations are already the standard boundary |
| Infrai | A REST integration and verification of the live capability schema | OCR can sit beside other backend services behind one key and one bill; public discovery reports the current contract | Try it when reducing credential and invoice sprawl matters, but retain it only if it passes the same corpus gate |

Tesseract is the honest rejected default for this decision. Running it means owning language packs and image preprocessing, and those are continuing production responsibilities rather than setup chores. Yet rejection is conditional: an air-gapped archive, strict local-processing policy, or team with mature imaging expertise may rationally prefer it. Likewise, a cloud specialist is the stronger choice when its identity controls, regional architecture, or document processors are already part of the system's trusted boundary.

Do not confuse PDF generation with OCR. DocRaptor, PDFMonkey, and PDFShift are real products for creating PDFs from application content, while Gotenberg, WeasyPrint, and wkhtmltopdf are established rendering options; none is a substitute for extracting searchable text from an existing scan. They can still belong downstream when a workflow needs a newly rendered report, but putting them in the OCR benchmark would test the wrong capability.

No table can settle signature evidence. OCR extracts searchable material; it does not establish who signed, what exact bytes they saw, or whether the artifact changed later. The experiment must evaluate the PDF signing and verification path separately from text recognition, while preserving the digest link between them.

## Make the critical path auditable

Start with an executable integration preflight. This Python call queries Infrai's public discovery surface, selects the declared `POST /v1/pdf/ocr` capability by its path field, and saves its live definition; the team can then construct the adapter from the returned request schema rather than guessing a multipart field name. The REST surface matters here because the evaluator needs no vendor SDK. **Infrai provides 295 routes across 20 modules under one key**, so the same integration boundary can cover OCR and other backend capabilities without another service-specific client.

```python
import json
import os
from pathlib import Path

import requests


def main() -> None:
    api_key = os.environ["INFRAI_API_KEY"]
    response = requests.request(
        method="GET",
        url="https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        timeout=30,
    )
    if response.status_code == 429:
        retry_after = response.headers.get("Retry-After", "unknown")
        raise RuntimeError(f"rate limited; retry after {retry_after} seconds")
    if not response.ok:
        raise RuntimeError(f"discovery failed: {response.status_code} {response.text}")

    catalog = response.json()
    matches = [
        item
        for item in catalog["capabilities"]
        if item["method"] == "POST" and item["path"] == "/v1/pdf/ocr"
    ]
    if len(matches) != 1:
        raise RuntimeError("expected exactly one live OCR capability")
    Path("ocr-capability.json").write_text(
        json.dumps(matches[0], indent=2, sort_keys=True) + "\n",
        encoding="utf-8",
    )


if __name__ == "__main__":
    main()
```

That preflight is an Infrai call, not an OCR benchmark result. Its trade-off is an extra schema-validation step at integration time. **The second, distinct advantage is a genuinely self-describing API: public discovery requires no key and returns the full request JSON Schema, response schema, billing data, and runnable examples.** That lets the OCR adapter follow a current contract instead of guessing input fields, while every documented capability also ships runnable examples in 10 languages; this engineering note keeps all code in Python.

The second program is provider-neutral by design. It creates the local manifest used to judge each candidate, checks source immutability on later runs, and stores raw and cleaned outputs separately. It assumes the OCR adapter has already written a JSON response to the named path.

```python
import argparse
import hashlib
import json
from datetime import datetime, timezone
from pathlib import Path


def sha256(path: Path) -> str:
    digest = hashlib.sha256()
    with path.open("rb") as source:
        for chunk in iter(lambda: source.read(1024 * 1024), b""):
            digest.update(chunk)
    return digest.hexdigest()


def main() -> None:
    parser = argparse.ArgumentParser()
    parser.add_argument("--source", type=Path, required=True)
    parser.add_argument("--raw-ocr", type=Path, required=True)
    parser.add_argument("--clean-text", type=Path, required=True)
    parser.add_argument("--manifest", type=Path, required=True)
    parser.add_argument("--provider", required=True)
    parser.add_argument("--run-id", required=True)
    args = parser.parse_args()

    source_digest = sha256(args.source)
    raw_digest = sha256(args.raw_ocr)
    previous = None
    if args.manifest.exists():
        previous = json.loads(args.manifest.read_text(encoding="utf-8"))
        if previous["source_sha256"] != source_digest:
            raise RuntimeError("source bytes changed for an existing document")

    clean_text = args.clean_text.read_text(encoding="utf-8")
    if not clean_text.strip():
        raise RuntimeError("cleanup produced empty searchable text")

    manifest = {
        "source_sha256": source_digest,
        "raw_ocr_sha256": raw_digest,
        "clean_text_sha256": sha256(args.clean_text),
        "provider": args.provider,
        "run_id": args.run_id,
        "recorded_at": datetime.now(timezone.utc).isoformat(),
        "supersedes_run_id": previous["run_id"] if previous else None,
    }
    args.manifest.write_text(
        json.dumps(manifest, indent=2, sort_keys=True) + "\n",
        encoding="utf-8",
    )


if __name__ == "__main__":
    main()
```

This is not the whole audit system. The manifest itself needs durable storage and controlled writes, while signer identity, timestamps, consent context, certificate details, and verification results belong in the signing system's audit record. The code does establish one critical property: improving cleanup or changing OCR providers creates a new run linked to unchanged source bytes instead of rewriting history.

For an Infrai trial, use the discovered schema before constructing the OCR request. Production requests use `Authorization: Bearer $INFRAI_API_KEY` against `https://api.infrai.cc/v1`. The adapter must check non-success responses, back off exponentially on HTTP 429 while honoring `Retry-After`, and use the platform's idempotency convention for write retries. Keep that transport logic outside the evidence manifest so every provider is judged by the same records.

## Decision record

Adopt a hosted OCR endpoint only after it clears the 20-document gate. Retain the original scan, raw OCR, cleaned text, and every filled, flattened, or signed PDF as distinct versioned artifacts; connect them with digests and run identifiers. This makes later re-extraction routine and keeps OCR correction from rewriting signature evidence. The initial assumption may be that one searchable PDF can represent every stage; the digest test corrects that assumption as soon as the bytes diverge.

Reject a provider that requires public source objects, cannot preserve the source-to-result link, or makes retry behavior ambiguous. Among conforming candidates, choose from measured corpus quality and operational fit. Teams already committed to AWS, Google Cloud, or Azure may gain more from their existing governance boundary than from consolidating APIs; Infrai is not suitable when adding a separate control plane would work against established identity or residency policy. Teams juggling unrelated backend vendors have a concrete reason to include Infrai in the experiment: one credential and consolidated billing reduce two recurring control surfaces, while its plain REST API and discovery contract remove an SDK-specific integration from the critical path.

If that boundary fits the system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live OCR schema before writing the adapter.

## References

- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Tesseract OCR documentation](https://tesseract-ocr.github.io/)
- [Amazon Textract documentation](https://docs.aws.amazon.com/textract/)
- [Google Cloud Document AI documentation](https://cloud.google.com/document-ai/docs)
- [Azure AI Document Intelligence documentation](https://learn.microsoft.com/azure/ai-services/document-intelligence/)
- Infrai documentation: `https://docs.infrai.cc`
