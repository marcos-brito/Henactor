<script lang="ts">
    import { commands, type Entry, type ListColumn, type SystemTime } from "$lib/bindings";
    import EntryIcon from "./entry-icon.svelte";
    import prettyBytes from "pretty-bytes";
    import { linkIsBroken } from "$lib/utils";
    import { open } from "./entry-actions.svelte";
    import mime from "mime";
    import EntryActions from "./entry-actions.svelte";
    import { trucate } from "$lib/utils";
    import { configManager, i18n } from "$lib";

    let { entry, columns }: { entry: Entry; columns: Array<ListColumn> } = $props();

    let ref = $state<HTMLTableRowElement>();

    function formatSystemTime(time: SystemTime): string {
        //@ts-ignore
        const date = new Date(time.secs_since_epoch * 1000);
        return date.toLocaleString(configManager.config.options.lang, {
            year: "numeric",
            day: "numeric",
            month: "short",
            hour: "numeric",
            minute: "numeric",
        });
    }

    async function resolveEntrySize(entry: Entry): Promise<string> {
        if (entry.entry_type == "Directory") {
            const entries = await commands.list(entry.path);
            return i18n.t("details.dirItem", { ns: "ui", count: entries.length });
        }

        return prettyBytes(entry.details.size);
    }

    async function mapColumn(column: ListColumn): Promise<string> {
        switch (column) {
            case "Kind":
                return i18n.t(`ui:details.kind.${entry.entry_type.toLowerCase()}`);
            case "Size":
                if (entry.entry_type == "Symlink" && !(await linkIsBroken(entry))) {
                    const target = await commands.findLinkTarget(entry.path);
                    return await resolveEntrySize(target);
                }
                return await resolveEntrySize(entry);
            case "Accessed":
                return entry.details.accessed ? formatSystemTime(entry.details.accessed) : "---";
            case "Modified":
                return entry.details.modified ? formatSystemTime(entry.details.modified) : "---";
            case "Created":
                return entry.details.created ? formatSystemTime(entry.details.created) : "---";
            case "DetailedKind":
                return mime.getType(entry.path) || "---";
        }
    }
</script>

{#if ref}
    <EntryActions trigger={ref} {entry} />
{/if}
<tr
    bind:this={ref}
    tabindex="0"
    ondblclick={async () => await open(entry)}
    class="entry hover rounded-btn border-none text-base outline-2 outline-current focus:outline"
>
    <td>
        <EntryIcon {entry} />
    </td>
    <td>
        {#if entry.entry_type == "Symlink"}
            {#await linkIsBroken(entry) then isBroken}
                {#if isBroken}
                    <p>eita</p>
                {/if}
            {/await}
        {/if}
        <p class="entry-name">
            {trucate(entry.name, configManager.config.options.truncation_limit)}
        </p>
    </td>
    {#each columns as column}
        <td>
            {#await mapColumn(column)}
                <p class="text-sm text-nowrap">---</p>
            {:then detail}
                <p class="text-sm text-nowrap opacity-70">{detail}</p>
            {/await}
        </td>
    {/each}
</tr>
