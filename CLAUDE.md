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
- Relation notations between code concepts use one canonical form, in code: memory-ordering pairs are `load→store`. Convert other spellings (hyphen, `->`) to it. This counts as an explicit rule for editing the assistant's text (see "The words are untouchable").
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

## Verifying a formatted transcript

- Compare word sequences of the original (`git show HEAD:<file>`) and the result, ignoring whitespace, punctuation and added markup (fences, bubble HTML, `**`, backticks, `Q:`/`A:` prefixes). Content lines can also start with `A:` (e.g. `A: revision 4001`); strip the prefix only on an answer's first line.
- Report every difference. The only expected ones are the typo fixes, the long-dash swaps and the canonical-notation conversions.
- The number of fence lines must be even.
- Preview the render locally (marked + github-markdown-css, served over http in the browser pane) before saying it looks right.

## Workflow

- "Check" / "verify" means report only. Don't edit, revert or fix anything you find until the user says so.
- Overwrite the transcript in place (the original is in git). Don't commit or push unless asked.
