<script lang="ts">
    import { t } from "../i18n";
    import { onMount } from "svelte";
    import { Button } from "@/lib/components/ui/button";
    import { Textarea } from "@/lib/components/ui/textarea";
    import type { QuizService } from "../quiz/service";
    import type { Question } from "@/types/quiz";
    import type { ReviewDraft } from "../quiz/review-workflow";
    import { validateReviewQuestion } from "../quiz/review-workflow";
    import { renderMarkdownLatex } from "./latex";
    let {
        service,
        initial = [],
        ondone,
    }: { service: QuizService; initial?: ReviewDraft[]; ondone?: (questions: Question[]) => void } = $props();
    let rows = $state<{ draft: ReviewDraft; json: string; options: string; answer: string }[]>([]);
    let error = $state("");
    let busy = $state(false);
    let message = $state("");
    function load(drafts: ReviewDraft[]) {
        rows = drafts.map((draft) => ({
            draft,
            json: JSON.stringify(draft.question, null, 2),
            options: "options" in draft.question ? draft.question.options.join("\n") : "",
            answer:
                draft.question.type === "subjective"
                    ? draft.question.reference_answer
                    : JSON.stringify(draft.question.answer),
        }));
    }
    function build(row: (typeof rows)[number]): Question {
        const q = { ...row.draft.question } as unknown as Record<string, unknown>;
        if (q.type === "single_choice" || q.type === "multiple_choice") q.options = row.options.split("\n");
        if (q.type === "subjective") q.reference_answer = row.answer;
        else q.answer = JSON.parse(row.answer);
        return validateReviewQuestion(q as unknown as Question);
    }
    onMount(() => {
        if (initial.length) load(JSON.parse(JSON.stringify(initial)));
        else
            void service.workflow
                ?.snapshot()
                .then((s) => load(s.pending))
                .catch((e) => (error = String(e)));
    });
    async function submit() {
        busy = true;
        error = "";
        try {
            const drafts = rows.map((row) => ({
                ...row.draft,
                question: row.draft.action === "delete" ? row.draft.question : build(row),
            }));
            const accepted = await service.workflow!.commit(JSON.parse(JSON.stringify(drafts)));
            message = `已通过 ${accepted.length} 题；跳过的题目保留在待定区。`;
            load((await service.workflow!.snapshot()).pending);
            ondone?.(accepted);
        } catch (e) {
            error = String(e);
        } finally {
            busy = false;
        }
    }
</script>

<section class="flex flex-col gap-4 p-4" aria-label={t("workflow.ReviewQueue.22")}>
    <header>
        <h2 class="text-lg font-semibold">{t("workflow.ReviewQueue.1")}{rows.length} 题</h2>
        <p class="text-muted-foreground text-sm">{t("workflow.ReviewQueue.2")}</p>
    </header>
    {#each rows as row (row.draft.id)}
        <article class="rounded-lg border p-4 flex flex-col gap-3">
            <div class="rg-markdown">{@html renderMarkdownLatex(row.draft.original.question)}</div>
            <label
                >{t("workflow.ReviewQueue.3")}<Textarea
                    aria-label={t("workflow.ReviewQueue.23")}
                    bind:value={row.draft.question.question}
                    disabled={busy}
                /></label
            >
            <label
                >{t("workflow.ReviewQueue.4")}<select
                    aria-label={t("workflow.ReviewQueue.24")}
                    bind:value={row.draft.question.type}
                    disabled={busy}
                    ><option value="single_choice">{t("workflow.ReviewQueue.5")}</option><option value="multiple_choice"
                        >{t("workflow.ReviewQueue.6")}</option
                    ><option value="fill_blank">{t("workflow.ReviewQueue.7")}</option><option value="true_false"
                        >{t("workflow.ReviewQueue.8")}</option
                    ><option value="subjective">{t("workflow.ReviewQueue.9")}</option><option value="coding"
                        >{t("workflow.ReviewQueue.10")}</option
                    ></select
                ></label
            >
            {#if row.draft.question.type === "single_choice" || row.draft.question.type === "multiple_choice"}<label
                    >{t("workflow.ReviewQueue.11")}<Textarea
                        aria-label={t("workflow.ReviewQueue.25")}
                        bind:value={row.options}
                    /></label
                >{/if}
            <label
                >{t("workflow.ReviewQueue.12")}{row.draft.question.type === "subjective"
                    ? "（参考答案文本）"
                    : '（JSON：单选 0，多选 [0,1]，判断 true，填空 [["答案"]]，编程用 JSON 字符串）'}<Textarea
                    aria-label={t("workflow.ReviewQueue.26")}
                    bind:value={row.answer}
                    disabled={busy}
                /></label
            >
            <label
                >{t("workflow.ReviewQueue.13")}<input
                    class="border rounded p-2 w-full"
                    aria-label={t("workflow.ReviewQueue.27")}
                    value={row.draft.question.tags?.join("，") ?? ""}
                    oninput={(e) =>
                        (row.draft.question.tags = e.currentTarget.value
                            .split(/[,，]/)
                            .map((s) => s.trim())
                            .filter(Boolean))}
                    disabled={busy}
                /></label
            >
            <label
                >{t("workflow.ReviewQueue.14")}<Textarea
                    aria-label={t("workflow.ReviewQueue.28")}
                    bind:value={row.draft.question.explanation}
                    disabled={busy}
                /></label
            >
            <details>
                <summary>{t("workflow.ReviewQueue.15")}</summary><Textarea
                    aria-label={t("workflow.ReviewQueue.29")}
                    class="min-h-48 font-mono text-xs"
                    bind:value={row.json}
                /><Button
                    variant="outline"
                    onclick={() => {
                        try {
                            const q = validateReviewQuestion(JSON.parse(row.json));
                            row.draft.question = q;
                            row.options = "options" in q ? q.options.join("\n") : "";
                            row.answer = q.type === "subjective" ? q.reference_answer : JSON.stringify(q.answer);
                            error = "";
                        } catch (e) {
                            error = String(e);
                        }
                    }}>{t("workflow.ReviewQueue.16")}</Button
                >
            </details>
            <label
                >{t("workflow.ReviewQueue.17")}<select
                    aria-label={t("workflow.ReviewQueue.30")}
                    bind:value={row.draft.action}
                    disabled={busy}
                    ><option value="pass">{t("workflow.ReviewQueue.18")}</option><option value="skip"
                        >{t("workflow.ReviewQueue.19")}</option
                    ><option value="delete">{t("workflow.ReviewQueue.20")}</option></select
                ></label
            >
            <label class="text-sm"
                >{t("workflow.ReviewQueue.21")}<Textarea
                    aria-label={t("workflow.ReviewQueue.31")}
                    bind:value={row.draft.reason}
                    disabled={busy}
                /></label
            >
        </article>
    {/each}
    {#if error}<p role="alert" class="text-destructive">{error}</p>{/if}
    {#if message}<p role="status">{message}</p>{/if}
    <Button onclick={submit} disabled={busy || !rows.length}>{busy ? "保存中…" : "确认审核并入库"}</Button>
</section>
