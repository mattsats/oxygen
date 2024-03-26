<script lang="ts">
  import { base } from "$app/paths";
  import { page } from '$app/stores';
  import { goto } from "$app/navigation";
  import makeEvent from "$lib/helpers/eventMaker";
  import { FAQ } from "$lib/stores/nostrocket_state/types";
  import { currentUser } from "$lib/stores/hot_resources/current-user";
  import { consensusTipState } from "$lib/stores/nostrocket_state/master_state";
  import { derived } from "svelte/store";
  import CommentUser from "../../components/comments/CommentUser.svelte";
  import { NDKEvent, NDKPrivateKeySigner } from "@nostr-dev-kit/ndk";
  import {
    TextArea,
    Button,
    Checkbox
  } from "carbon-components-svelte";
  import { ConvertToCloud } from "carbon-icons-svelte";
  import { InlineLoading } from "carbon-components-svelte";
  import { nostrocketIgnitionEvent, simulateEvents } from "../../settings";

  let isAnon = $currentUser?false:true;
  let anonSigner = NDKPrivateKeySigner.generate();
  let newQ = true;

  export let newFAQ: FAQ = new FAQ();

  export let rocketID: string = nostrocketIgnitionEvent;

  let faqs = derived(consensusTipState, ($cts) => {
    let rocket = $cts.RocketMap.get(nostrocketIgnitionEvent);
    if (rocket) {
      return rocket.FAQ;
    }
    return new Map<String, FAQ>();
  });

  let rocket = derived(consensusTipState, ($cts) => {
    return $cts.RocketMap.get(rocketID || newFAQ.RocketID);
  });

  function publish() {
    let e = makeEvent({
      kind: 1122,
      rocket: $rocket?.UID || newFAQ.RocketID,
    });
    e.tags.push(["text", newFAQ.Question, "question"]);
    e.tags.push(["new"]);
    e.tags.push([
      "alt",
      "[ " + newFAQ.Question + " ]",
    ]);
    e.author = $currentUser!;
    e.content =
      "[ " +
      newFAQ.Question +
      " ]" +
      " read more using a Nostrocket client.";
    e.tags.push(["r", $page.url.toString()])


    if (!simulateEvents) {
      if (!isAnon) {
        // TODO: PUBLISH
        // e.publish()
        //   .then((x) => {
        //     console.log(e.rawEvent(), x);
        //     //goto(`${base}/FAQ/${newFAQ.UID?.length == 64?newFAQ.UID:e.id}`);
        //   })
        //   .catch((err) => {
        //     console.log(e);
        //     throw new Error("failed to publish Problem event. " + err);
        //   });
      } else {
        let ee = new NDKEvent(undefined, e.rawEvent());
        ee.sign(anonSigner).then(() => {
          console.log(ee.rawEvent());
        })
        // TODO: PUBLISH
      }
    } else {
      e.sign().then(() => {
        console.log(e.rawEvent());
      });
    }
  }

  </script>

<div style="width: 100%;padding:2px;margin-bottom:10%;">
    <h6>Help! I have a question...</h6>
    <hr />
    {#if newQ}
      <TextArea light
        helperText={""}
        maxlength={100}
        placeholder="Type your question here..."
        bind:value={newFAQ.Question}
      />
      {#if $currentUser}
        Ask
        {#if !isAnon} as
          {#if $currentUser}
            <CommentUser pubkey={$currentUser.pubkey} />
          {/if}
        {:else}
          anonymously
        {/if}
        <Checkbox
          labelText="Anonymous"
          bind:checked={isAnon}
        />
      {:else}
        {#if isAnon}
          Ask anonymously
        {/if}
      {/if}
      <br>
      <Button
      disabled={newFAQ.Question.length < 10}
      icon={ConvertToCloud}
      on:click={() => {
        publish();
        //newQ = false; // TODO: FIX BUG???
      }}>Ask</Button
      >
    {/if}
    {#if !newQ}
      Thanks for asking! We will get back to you soon.
    {/if}
  </div>
  <div style="width: 100%;padding:2px;">
    <h6 on:click={() => goto(`${base}/FAQ/`)}>Frequently asked questions:</h6>
    <hr />
    {#if !$faqs || $faqs.size == 0}<InlineLoading /> WAITING FOR EVENTS{/if}

    <ul>
    {#each $faqs as [id, faq]}
      <li
       on:click={()=> goto(`${base}/FAQ/${faq.UID}`)}>{faq.Question}</li>
    {/each}
    </ul>
  </div>
