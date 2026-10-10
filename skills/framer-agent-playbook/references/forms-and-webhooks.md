# Forms and webhooks

How a Framer form behaves when it posts to a webhook, learned building a pipeline where a Framer
form posts to a Val Town val that creates a CMS item through `framer-api` (2026-09-25 and 26).

- **Form button variants need the button linked first.** `formButtonPendingVariant` and
  `formButtonSuccessVariant` on the submit instance are refused until the form carries
  `formSubmitButtonId`; set that, then the variants, in a later call. New inputs come with a grey
  1px border, a 10px radius and a blue focus border: `border="null" radius="0px"` and the
  `formInputFocused*` attributes restyle them (2026-09-27).
- **Required fields in a native form use the browser's own check:** an empty submit focuses the
  first one (2026-10-04).
- **Forms in a component (2026-10-04).** `DUPE <formId> newId="x" parent="<variant>"`
  copies a page's form into a new component's variant, inputs and button included, and
  `formSubmitButtonId` follows the copy. `CREATE_VARIANT` then gives Sending and Sent states
  (`opacity`, `pointerEvents="none"`, `visible`). **The form-state to component-variant mapping is
  not in the API**: no DSL attribute, and a form configured in the editor serializes without it, so the
  editor sets Pending, Success and Error on the form. Radios with `formBooleanInputValue` unset
  rendered the last radio of a group checked on the published site; set `"false"` on every
  checkbox and radio. `formInputCheckedBorder="6px solid <ink>"` with a white
  `formInputCheckedFill` draws the checked dot.
- **The agent cannot set a form's destination** (webhook, email). No DSL attribute exists and the
  raw form node carries no such property. Build the form with the agent, then set the destination
  in the editor (2026-09-25).
- **The button says "Sent" before the webhook runs.** The browser posts to
  `api.framer.com/forms/v1/forms/<id>/submit`; Framer answers, the submit button switches to its
  Success variant, and only then does Framer call the webhook. The page cannot see the webhook's
  answer. To give visitors feedback, the webhook has to expose its verdict some other way (for
  example a `GET /status` endpoint that the page polls).
- **Any non-2xx webhook response sends the site owner an email** with the submission's contents.
  So a webhook must accept everything the browser's own checks accept, and answer 200 when it
  drops something on purpose (spam, rate limits), or the owner's inbox fills. Use the WHATWG
  `type="email"` rule for email: it accepts `test@test`, and a stricter regex caused a real 400.
- **The webhook request is signed:** `Framer-Signature: sha256=<hex HMAC-SHA256(secret, rawBody +
  submissionId)>` with the submission id in `Framer-Webhook-Submission-Id`. The body is JSON keyed
  by input names. The secret is set in the form's webhook settings (32+ characters) and cannot be
  viewed again.
- **Framer's form already defends itself a little.** Each browser post carries a proof-of-work
  header (`Framer-POW`) computed by the site script, and the form renders its own hidden honeypot
  inputs (`website`, `company`, `subject`, `title` and more).
- **To add a computed value to a submission**, add a real hidden input (`formInputHidden`, with a
  `formInputName`) and fill it from an override in a capture-phase `submit` listener, which
  runs before Framer's own handler reads the form.
- **An override can watch for a successful submission** by wrapping `window.fetch` and matching a
  POST to `api.framer.com/forms/` with `res.ok`. Framer's form script calls the global `fetch`.

- **Native form inputs start 100px tall with a grey border.** Set the height, then
  `borderColor`, `borderStyle` and per-side `borderTop="0px" borderBottom="1px"`; `border="null"`
  followed by `borderBottom` did not apply (2026-10-08).
