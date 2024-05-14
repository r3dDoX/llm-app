<script lang="ts">
  import SvelteMarkdown from 'svelte-markdown'
  import * as webllm from "@mlc-ai/web-llm";
  import type {ChatCompletionMessageParam, EngineInterface} from "@mlc-ai/web-llm";

  let engineReport: webllm.InitProgressReport;
  let prompt: string | undefined;
  let response: string | undefined;
  let chatHistory: Array<ChatCompletionMessageParam> = [];

  async function setupEngine(): Promise<EngineInterface> {
    const selectedModel = "Llama-3-8B-Instruct-q4f32_1";
    return webllm.CreateEngine(
      selectedModel,
      {initProgressCallback: (report) => engineReport = report}
    );
  }

  const engine = setupEngine();

  async function handlePrompt() {
    if (!prompt) {
      return;
    }

    chatHistory = [...chatHistory, {role: 'user', content: prompt}];
    prompt = undefined;
    const completion = await (await engine).chat.completions.create({
      stream: true,
      messages: chatHistory,
    });

    let currentMessage = '';
    for await (const chunk of completion) {
      const currentDelta = chunk.choices[0].delta.content;
      if (currentDelta) {
        currentMessage += currentDelta;
        response = currentMessage;
        window.scrollTo(0, document.body.scrollHeight);
      }
    }
    chatHistory = [...chatHistory, {role: 'assistant', content: currentMessage}];
    response = undefined;
  }

  function handlePromptKeyDown(event: KeyboardEvent) {
    if (event.key === 'Enter') {
      handlePrompt();
    }
  }
</script>

<style>
    :global(body) {
        margin: 0;
        padding: 0;
        background-color: papayawhip;
        font-family: "Inter", sans-serif;
        font-optical-sizing: auto;
    }

    h1, p {
        padding: 0 16px;
    }

    .conversation {
        display: flex;
        flex-direction: column;
        margin: 20px 0;
    }

    .conversation > div {
        align-self: flex-start;
        border-radius: 8px;
        padding: 0 16px;
        margin: 8px;
        background-color: cadetblue;
    }

    .conversation > div.user {
        color: white;
        align-self: flex-end;
        background-color: darkslateblue;
    }

    .prompt {
        display: flex;
        align-items: center;
        padding: 8px 16px;
        gap: 20px;
        background-color: darkslateblue;
        position: sticky;
        bottom: 0;
        right: 0;
        left: 0;
    }
</style>

<h1>Web LLM Test</h1>
<p>
    {engineReport?.text}
</p>
<div class="conversation">
    {#each chatHistory as message}
        <div class="{message.role}">
            <SvelteMarkdown source={message.content}></SvelteMarkdown>
        </div>
    {/each}
    {#if response}
        <div>
            <SvelteMarkdown source={response}></SvelteMarkdown>
        </div>
    {/if}
</div>
{#if engineReport?.progress >= 1}
    <div class="prompt">
        <textarea bind:value={prompt} on:keydown={handlePromptKeyDown} cols="50" rows="3"></textarea>
        <button on:click={handlePrompt}>Generate!</button>
    </div>
{/if}