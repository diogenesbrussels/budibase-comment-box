<script>
  import { getContext, afterUpdate } from "svelte";
  import Avatar from "./Avatar.svelte";
  import Comment from "./Comment.svelte";

  export let table;
  export let column;
  export let rowId;
  export let itemName, itemNamePlural, messagePlaceholder, buttonText;
  export let dateFormat, dateRelative;

  const { styleable, API, authStore, notificationStore } = getContext("sdk");
  const component = getContext("component");
  const separator = "|";

  let comments = [];
  let text;
  let commentContainer;
  let lastCommentCount = 0;

  $: table, column, rowId, loadComments();
  $: itemName, itemNamePlural, messagePlaceholder, buttonText;
  $: dateFormat, dateRelative;
  $: currentName = `${$authStore.firstName || ""} ${
    $authStore.lastName || ""
  }`.trim();

  const getRow = async () => {
    if (!table?.tableId || !rowId || !column) {
      return null;
    }
    return await API.fetchRow({ rowId, tableId: table.tableId });
  };

  const getComments = async () => {
    const row = await getRow();
    const value = row?.[column];
    if (typeof value !== "string" || !value?.length) {
      return [];
    }
    return (
      value
        .split(separator)
        .map(atob)
        .map(escape)
        .map(decodeURIComponent)
        .map(JSON.parse) || []
    );
  };

  const loadComments = async () => {
    try {
      comments = await getComments();
    } catch (error) {
      comments = [];
    }
  };

  const saveComments = async (comments) => {
    if (!Array.isArray(comments)) {
      return null;
    }
    try {
      // Encode the array of comments
      const encoded = comments
        .map(JSON.stringify)
        .map(encodeURIComponent)
        .map(unescape)
        .map(btoa)
        .join(separator);

      // Update row
      const row = await getRow();
      await API.saveRow({
        ...row,
        [column]: encoded,
      });

      // Refresh from the server to ensure we're consistent, and to update UI
      await loadComments();
    } catch (error) {
      notificationStore.actions.error("Failed to save " + itemNamePlural);
      console.error(error);
    }
  };

  const addComment = async () => {
    if (!text?.trim().length) {
      return;
    }
    try {
      // Clear state
      const message = text.trim();
      text = null;

      // Create and save new comment
      const existingComments = await getComments();
      const newComment = {
        message,
        email: $authStore.email,
        name: currentName,
        timestamp: Date.now(),
      };
      await saveComments([...existingComments, newComment]);
    } catch (error) {
      notificationStore.actions.error("Failed to add " + itemName);
      console.error(error);
    }
  };

  const deleteComment = async (comment) => {
    if (!comment?.timestamp) {
      return;
    }
    try {
      // Fetch existing comments and filter out this one
      const existingComments = await getComments();
      await saveComments(
        existingComments.filter((x) => x.timestamp !== comment.timestamp)
      );
    } catch (error) {
      notificationStore.actions.error("Failed to delete " + itemName);
      console.error(error);
    }
  };

  const handleKeyPress = (e) => {
    if (e.key === "Enter") {
      e.preventDefault();
      e.stopPropagation();

      if (!e.shiftKey) {
        addComment();
      } else {
        e.target.value += "\n";
      }
    }
  };

  afterUpdate(() => {
    if (comments.length !== lastCommentCount) {
      commentContainer.scrollTop = commentContainer.scrollHeight;
      lastCommentCount = comments.length;
    }
  });
</script>

<div use:styleable={$component.styles} class="container">
  <div class="title">
    {comments.length}
    {comments.length <= 1 ? itemName : itemNamePlural}
  </div>
  <div class="comments" bind:this={commentContainer}>
    {#each comments as comment (comment.timestamp)}
      <Comment
        {comment}
        destroy={() => deleteComment(comment)}
        {dateFormat}
        {dateRelative}
      />
    {/each}
  </div>
  <div class="form">
    <Avatar name={currentName} email={$authStore.email} />
    <textarea
      on:keypress={handleKeyPress}
      bind:value={text}
      rows="2"
      placeholder={messagePlaceholder}
    />
    <div />
    <div class="button">
      <button
        class="spectrum-Button spectrum-Button--sizeM spectrum-Button--cta"
        on:click={addComment}>{buttonText}</button
      >
    </div>
  </div>
</div>

<style>
  .container {
    background: var(--spectrum-global-color-gray-50);
    padding: 16px;
    border-radius: 4px;
    border: 1px solid var(--spectrum-global-color-gray-200);
    display: flex;
    flex-direction: column;
    gap: 16px;
    font-family: var(--font-sans);
  }
  .title {
    font-weight: bold;
    font-size: 16px;
  }
  .comments {
    background: var(--spectrum-global-color-gray-100);
    min-height: 100px;
    max-height: 320px;
    padding: 12px;
    border-radius: 4px;
    border: 1px solid var(--spectrum-global-color-gray-200);
    overflow-y: auto;
    display: flex;
    flex-direction: column;
    gap: 12px;
  }
  .form {
    display: grid;
    grid-template-columns: auto 1fr;
    gap: 12px;
  }
  textarea {
    flex: 1 1 auto;
    border: 1px solid var(--spectrum-global-color-gray-400);
    padding: 12px;
    border-radius: 4px;
    font-size: 14px;
    font-family: var(--font-sans);
    background: var(--spectrum-global-color-gray-50);
    color: var(--text-color);
    transition: border-color 130ms ease-out;
  }
  textarea:focus {
    outline: none;
    border-color: var(--spectrum-global-color-blue-600);
  }
  .button {
    display: flex;
    justify-content: flex-end;
  }
</style>
