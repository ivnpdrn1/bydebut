## 27. Pegar código `generate-version/index.ts`

Copia y pega esto en:

```text
Edge Functions
→ generate-version
→ index.ts
```

```ts
import { serve } from "https://deno.land/std@0.224.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2";

const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
};

serve(async (req) => {
  if (req.method === "OPTIONS") {
    return new Response("ok", { headers: corsHeaders });
  }

  try {
    const supabaseUrl = Deno.env.get("SUPABASE_URL");
    const serviceRoleKey = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY");
    const geminiApiKey = Deno.env.get("GEMINI_API_KEY");

    if (!supabaseUrl || !serviceRoleKey || !geminiApiKey) {
      throw new Error("Missing environment variables.");
    }

    const supabase = createClient(supabaseUrl, serviceRoleKey);

    const body = await req.json();

    const {
      user_id,
      book_id,
      chapter_id,
      scene_id,
      generation_mode,
      variables,
      prompt_extra,
      visibility
    } = body;

    if (!user_id || !book_id) {
      throw new Error("user_id and book_id are required.");
    }

    const { data: profile, error: profileError } = await supabase
      .from("profiles")
      .select("id, credits_available")
      .eq("id", user_id)
      .single();

    if (profileError) throw profileError;

    if (!profile || profile.credits_available <= 0) {
      throw new Error("Not enough credits available.");
    }

    const { data: book, error: bookError } = await supabase
      .from("books")
      .select("*")
      .eq("id", book_id)
      .single();

    if (bookError) throw bookError;

    let chapter = null;
    let scene = null;

    if (chapter_id) {
      const { data, error } = await supabase
        .from("chapters")
        .select("*")
        .eq("id", chapter_id)
        .single();

      if (error) throw error;
      chapter = data;
    }

    if (scene_id) {
      const { data, error } = await supabase
        .from("scenes")
        .select("*")
        .eq("id", scene_id)
        .single();

      if (error) throw error;
      scene = data;
    }

    const variableText = Array.isArray(variables)
      ? variables
          .map((v) => `- ${v.name}: ${v.value} | weight: ${v.weight || 1}`)
          .join("\n")
      : "";

    const sourceText = scene?.content || chapter?.content || book?.description || "";

    const prompt = `
You are byDebut, an AI literary transformation engine.

Your task is to create a new literary version based on the original content and the selected narrative variables.

Book:
${book.title || ""}

Author:
${book.author_name || ""}

Genre:
${book.genre || ""}

Language:
${book.language || "en"}

Book Description:
${book.description || ""}

Chapter:
${chapter?.title || ""}

Chapter Summary:
${chapter?.summary || ""}

Scene:
${scene?.title || ""}

Scene Summary:
${scene?.summary || ""}

Generation Mode:
${generation_mode || "literary"}

Narrative Variables:
${variableText}

Additional User Instructions:
${prompt_extra || ""}

Original Content:
${sourceText}

Rules:
- Preserve the literary quality.
- Do not simply summarize.
- Create a transformed version.
- Keep the narrative coherent.
- Respect the selected variables.
- Make the result commercially interesting.
- Avoid explaining your process.
- Return only the generated literary content.
`;

    const geminiResponse = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-1.5-flash:generateContent?key=${geminiApiKey}`,
      {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
        },
        body: JSON.stringify({
          contents: [
            {
              parts: [
                {
                  text: prompt,
                },
              ],
            },
          ],
          generationConfig: {
            temperature: 0.85,
            topP: 0.95,
            maxOutputTokens: 4096,
          },
        }),
      }
    );

    if (!geminiResponse.ok) {
      const errorText = await geminiResponse.text();
      throw new Error(`Gemini error: ${errorText}`);
    }

    const geminiData = await geminiResponse.json();

    const generatedContent =
      geminiData?.candidates?.[0]?.content?.parts?.[0]?.text || "";

    if (!generatedContent) {
      throw new Error("Gemini returned empty generated content.");
    }

    const versionTitle = `Variation of ${chapter?.title || scene?.title || book.title}`;

    const { data: savedVersion, error: saveVersionError } = await supabase
      .from("story_versions")
      .insert({
        book_id,
        chapter_id: chapter_id || null,
        scene_id: scene_id || null,
        creator_id: user_id,
        title: versionTitle,
        prompt,
        generated_content: generatedContent,
        generation_mode: generation_mode || "literary",
        visibility: visibility || "private",
        status: "mvp",
        is_fork: false,
      })
      .select()
      .single();

    if (saveVersionError) throw saveVersionError;

    if (Array.isArray(variables)) {
      for (const v of variables) {
        const { data: variableRecord } = await supabase
          .from("narrative_variables")
          .select("id")
          .eq("name", v.name)
          .single();

        if (variableRecord?.id) {
          await supabase.from("version_variables").insert({
            version_id: savedVersion.id,
            variable_id: variableRecord.id,
            selected_value: v.value,
            weight: v.weight || 1,
          });
        }
      }
    }

    await supabase
      .from("profiles")
      .update({
        credits_available: profile.credits_available - 1,
      })
      .eq("id", user_id);

    await supabase.from("credit_transactions").insert({
      user_id,
      amount: -1,
      reason: "generate-version",
      related_version_id: savedVersion.id,
    });

    await supabase.from("ai_logs").insert({
      user_id,
      function_name: "generate-version",
      model_name: "gemini-1.5-flash",
      request_payload: {
        book_id,
        chapter_id,
        scene_id,
        generation_mode,
        variables,
        prompt_extra,
      },
      response_payload: {
        version_id: savedVersion.id,
      },
    });

    return new Response(
      JSON.stringify({
        success: true,
        version: savedVersion,
      }),
      {
        headers: {
          ...corsHeaders,
          "Content-Type": "application/json",
        },
      }
    );
  } catch (error) {
    return new Response(
      JSON.stringify({
        success: false,
        error: error.message,
      }),
      {
        status: 400,
        headers: {
          ...corsHeaders,
          "Content-Type": "application/json",
        },
      }
    );
  }
});
```

---

## 28. Probar `generate-version`

Usa este JSON de prueba:

```json
{
  "user_id": "AQUI_ID_DEL_USUARIO",
  "book_id": "AQUI_ID_DEL_BOOK",
  "chapter_id": "AQUI_ID_DEL_CHAPTER",
  "generation_mode": "literary",
  "variables": [
    {
      "name": "love",
      "value": "high",
      "weight": 1.2
    },
    {
      "name": "power",
      "value": "high",
      "weight": 1.5
    },
    {
      "name": "artificial_intelligence",
      "value": "medium",
      "weight": 1
    }
  ],
  "prompt_extra": "Make the story more emotional, cinematic, and commercially attractive.",
  "visibility": "private"
}
```

Cuando funcione, debe crear un registro en:

```text
story_versions
version_variables
credit_transactions
ai_logs
```

---

## 29. Crear Edge Function `review-version`

Ahora crea la siguiente función:

```text
Edge Functions
→ Create a new function
```

Nombre:

```text
review-version
```

El siguiente paso será:

```text
30. Pegar código review-version/index.ts
```
