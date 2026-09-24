# tech-notes

Personal tech notes. Many are raw copy-pastes of AI chats (e.g. ChatGPT) to be formatted into readable Markdown. Read on GitHub and in local previewers (VS Code, Obsidian).

## Formatting a chat transcript

Raw input: user turns start with `Q:`, assistant turns with `A:`. ASCII diagrams have lost their code formatting.

### The words are untouchable

- Keep every word letter-for-letter. Change only whitespace/indentation and the markup around the text.
- Fix obvious typos **only in the user's questions**. No rewording, added words, grammar, punctuation or capitalization fixes.
- Never edit the assistant's text.

### Character preferences

- Try not to use (unless explicitly requested and/or required for some clear reason) the "long" dash ('—'). Prefer the "regular" dash ('-') instead.

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
- Report every difference. The only expected ones are the typo fixes.
- The number of fence lines must be even.
- Preview the render locally (marked + github-markdown-css, served over http in the browser pane) before saying it looks right.

## Workflow

- "Check" / "verify" means report only. Don't edit, revert or fix anything you find until the user says so.
- Overwrite the transcript in place (the original is in git). Don't commit or push unless asked.
