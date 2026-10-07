# Comment Post button validation

Local browser review with disposable sample data, 7 October 2026. No production content is included.

In Safari, pasting a comment into an empty editor left Post disabled. Pressing another key could enable it. The application checks form validity after one animation frame, but Lexxy 0.9.32 dispatches `lexxy:change` before scheduling its own validity refresh for that frame. The form check can therefore run first and retain the previous empty state.

The fix waits one additional frame before reading the same form validity. It preserves required-field and attachment validation; it does not enable submission merely because a key was pressed.

## Before: Safari

Pasted content is present, but Post remains disabled.

![Safari before the fix](comment-safari-before.png)

## After: Safari

One paste enables Post with no extra key. Clicking Post successfully added the comment to the conversation and reset the composer to empty with Post disabled.

![Safari after the fix](comment-safari-after.png)

## Regression checks

- The single-update system test fails on the original code: Post stays disabled after inserting content.
- The same test passes with the fix, including clearing and reinserting text.
- Restoring a saved draft enables Post without typing.
- Ctrl+Enter successfully posts a comment.
- The existing attachment upload/Post/lightbox system test passes.
- Final Chromium system run: 4 tests, 28 assertions, no failures or errors.
- RuboCop passes for the new test file.

The full test suite and iOS Safari were not run. The fix was reviewed in desktop Safari and Chromium locally; it has not been deployed to the self-hosted production instance.
