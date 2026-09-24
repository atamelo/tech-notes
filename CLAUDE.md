# tech-notes

Personal tech notes. Many are raw copy-pastes of AI chats (e.g. ChatGPT) to be formatted into readable Markdown. Read on GitHub and in local previewers (VS Code, Obsidian).

## Formatting a chat transcript

Raw input: user turns start with `Q:`, assistant turns with `A:`. ASCII diagrams have lost their code formatting.

### The words are untouchable

- Keep every word letter-for-letter. Change only whitespace/indentation and the markup around the text.
- Fix obvious typos **only in the user's questions**. No rewording, added words, grammar, punctuation or capitalization fixes.
- Never edit the assistant's text (except when explicitly asked by the user or when there is an explicit rule below requiring that).

### Character preferences

- You should not allow (unless explicitly requested and/or required for some clear reason) the "long" dash ('—'). Prefer the "regular" dash ('-') instead. Applies to all text, including the assistant's; an unspaced `a—b` becomes `a - b`.

### Inline code

- Format a token as code if a reader could type it into code, a debugger or a man page and it would mean the same thing there: identifiers (functions, types, fields, variables), constants/flags/macros, instructions and registers, addresses and operands, literal values or expressions quoted from nearby code.
- Leave it plain when it's used as a noun for a component, product or concept rather than as that code token (e.g. "the X thread", "the Y subsystem"). Ask: would it still read correctly with the code formatting removed? If the answer is "yes, it's just a name", leave it.
- Don't format a term that appears in nearly every sentence of a file; the formatting stops carrying signal. Its literal code uses (`X=4`, `X-1`) still get formatted.
- Applies to answers, headings and questions; in a question bubble use `<code>…</code>`. Never touch code blocks, `<pre>`, links, or text that's already code.

### Questions → right-aligned chat bubble

No headings, `---` separators or TOC for questions: they continue the conversation, they aren't sections.

```html
<table
  style="margin: 32px 0 20px auto; max-width: 75%; border-collapse: separate; border: none;"
>
  <tr>
    <td
      style="background: #1e3f73; color: #ffffff; border: none; border-radius: 20px; padding: 12px 18px; line-height: 1.5;"
    >
      <strong style="font-size: 1.4em;">Q:</strong>&nbsp; The question text, on
      a single line.
    </td>
  </tr>
</table>
```

- No blank lines inside the table (a blank line ends the HTML block). HTML-escape `<` and `&` in the question.
- If GitHub strips `style`: the bubble degrades to a bordered cell with a bold **Q:**. That's accepted.

### Answers → plain Markdown after the bubble

- Drop the `A:` prefix.
- Put every ASCII diagram and every snippet (pseudo-code, timelines, key/value, messages) in a ` ```text ` block. Unfenced snippets are what breaks on GitHub: their lines collapse into one paragraph.
- Realign diagrams with whitespace so boxes and arrows line up. If you add or remove any drawing character, list each such change in your summary.
- The assistant's plain-line section titles → `###`. Plain-line enumerations → Markdown lists. Standalone takeaway sentences → `>` blockquote.
- Blank line before tables.
- File starts with `# Title` and one italic line saying what the conversation covers.

### Images from the chat

- **Placement:** put each image where the chat rendered it. Find the anchor in the text: a lead-in that now leads nowhere ("Let me visualize…", "once you have this chain"), a glued progress message, or the paragraph the image's content illustrates. Insert right after the anchor, before the next paragraph or heading. If no anchor is clear, ask.
- **Storage:** copy the file byte for byte into an `images/` folder next to the note, keeping its name, and reference it with a relative Markdown image. Take the alt text from the image's own title/description, or summarize what it shows. Don't embed SVG code or data URIs: GitHub strips both.
- **Never modify the original.** Anything that has to change (colors, background, size) goes into a new copy with a suffix (`name_dark.svg`), and the note links to the copy. If a copy invalidates embedded provenance (a signed content credential), remove that block from the copy only.
- **When asked, match the look the user saw.** Exported images can have a different theme baked in than the one displayed. Derive color changes from the source design system's documented light/dark mapping, not by eye, and map each color by its role (fill, border, title, subtitle, label, connector). Check the result against the user's screenshot by comparing rendered pixel colors.
- **Verify the render:** the image loads, sits between the intended neighbors, displays at a sensible size, and every color in it maps (fail on any unmapped one instead of guessing).
- **Report** which image files are new and untracked, so they get committed together with the note.

## Verifying a formatted transcript

- Compare word sequences of the original (`git show HEAD:<file>`) and the result, ignoring whitespace, punctuation and added markup (fences, bubble HTML, `**`, backticks, `Q:`/`A:` prefixes, inserted image lines). Content lines can also start with `A:` (e.g. `A: revision 4001`); strip the prefix only on an answer's first line.
- Report every difference. The only expected ones are the typo fixes and the long-dash swaps.
- The number of fence lines must be even.
- Preview the render locally (marked + github-markdown-css, served over http in the browser pane) before saying it looks right.

## Workflow

- "Check" / "verify" means report only. Don't edit, revert or fix anything you find until the user says so.
- Overwrite the transcript in place (the original is in git). Don't commit or push unless asked.
