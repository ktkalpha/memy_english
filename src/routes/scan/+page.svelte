<script module lang="ts">
    import { env } from "$env/dynamic/private";
    import {
        GoogleGenAI,
        createUserContent,
        createPartFromUri,
    } from "@google/genai";
    import { API_KEY } from "$env/static/private";
    // Gemini API 세팅
    const ai = new GoogleGenAI({
        apiKey: API_KEY,
    });
</script>

<script lang="ts">
    // 사진 json 변환
    async function image_to_json(): Promise<
        | {
              word: string;
              koreanMeaning: string;
              englishMeaning: string;
              exampleSentence: string;
          }[]
        | undefined
    > {
        // 파일 입력
        const fileInput = document.getElementById("docu") as HTMLInputElement;
        // 파일 업로드
        let selectedFile;
        if (fileInput.files && fileInput.files.length > 0) {
            selectedFile = await ai.files.upload({
                file: fileInput.files[0],
                config: { mimeType: "image/jpeg" },
            });
        } else {
            // 파일 업로드 실패
            return undefined;
        }
        // 파일 업로드 성공
        if (selectedFile?.uri && selectedFile?.mimeType) {
            // Gemini 2.0 Flash Lite 통해 이미지 처리
            // ! 분당 30 requests

            const response = await ai.models.generateContent({
                model: "gemini-2.0-flash-lite",
                contents: createUserContent([
                    createPartFromUri(selectedFile.uri, selectedFile.mimeType),
                    "Extract words from this image and output as JSON array of objects with keys: word, koreanMeaning, englishMeaning, exampleSentence. Output only the JSON array.",
                ]),
            });

            // response -> json parse
            const raw = response.text ?? "[]";
            // 코드 markdown 제거
            let cleaned = raw.trim();
            if (cleaned.startsWith("```")) {
                cleaned = cleaned
                    .replace(/^```(?:json)?\s*/, "")
                    .replace(/\s*```$/, "");
            }
            try {
                const entries: {
                    word: string;
                    koreanMeaning: string;
                    englishMeaning: string;
                    exampleSentence: string;
                }[] = JSON.parse(cleaned);

                return entries;
            } catch (error) {
                console.error(
                    "Failed to parse JSON:",
                    error,
                    "Raw response:",
                    raw,
                );
                return undefined; // JSON parse 에러 처리
            }
        }
        return undefined; // 파일 업로드 실패 처리
    }

    // 사진에서 단어 불러오기
    async function get_words_from_image() {
        words = (await image_to_json()) ?? [];
    }

    // localStroage에서 단어 불러오기
    function get_words_from_local(title: string) {
        const new_words_data = window.localStorage.getItem(title); // title을 key로 stringify된 단어장 데이터 불러옴
        if (new_words_data) {
            const new_words = JSON.parse(new_words_data); // parse
            if (new_words && confirm("저장하지 않은 단어장은 사라집니다.")) {
                // 유저 저장 확인
                words = new_words;
                return;
            }
        }
        console.error("해당 제목 데이터가 존재하지 않습니다: ", title);
    }

    function save_words() {
        const title = prompt("저장 데이터 이름: ");
        if (
            title &&
            confirm("이미 제목이 존재하는 경우 데이터는 덮어써집니다")
        ) {
            word_saves_list.push(title);
            window.localStorage.setItem(title, JSON.stringify(words));
            window.localStorage.setItem(
                "word_saves_list",
                JSON.stringify(word_saves_list),
            );
        }
    }

    let words: {
        word: string;
        koreanMeaning: string;
        englishMeaning: string;
        exampleSentence: string;
    }[] = $state([
        {
            word: "wow",
            koreanMeaning: "헉",
            englishMeaning: "",
            exampleSentence: "",
        },
    ]);
    let word_saves_list: string[] = $state([]);
    $effect(() => {
        word_saves_list = JSON.parse(
            window.localStorage.getItem("word_saves_list") ?? "[]",
        );
    });
</script>

<main>
    <input type="file" id="docu" name="docu" accept="image/jpeg" />
    <button onclick={save_words}>save</button>
    <button onclick={get_words_from_image}>image get word</button>
    <ul>
        {#each word_saves_list as save}
            <li>
                {save}<button
                    onclick={() => {
                        get_words_from_local(save);
                    }}>불러오기</button
                >
            </li>
        {/each}
    </ul>
    <p>result:</p>
    {#each words as word, index}
        <p>{index + 1}. {word.word}: {word.koreanMeaning}</p>
    {/each}
</main>
