
# AI-Powered-Cyberbullying-threat-detection-with-Mental-health-chatbot

This project proposes a technically robust, AI-powered solution that combines two key components: a cyberbullying detection engine powered by a transformer-based model (Toxic-BERT), and an integrated mental health chatbot built using BlenderBot.

---

## Library Installations

```bash
!pip install transformers datasets gradio --quiet
!pip install --upgrade datasets
!pip install --upgrade gradio
!pip install --upgrade transformers
```

## Import Libraries

```bash
import pandas as pd
import numpy as np
import re
import string
from datasets import Dataset
from transformers import BertTokenizer, BertForSequenceClassification, Trainer, TrainingArguments, pipeline
import gradio as gr
from sklearn.metrics import classification_report, accuracy_score
from transformers import AutoTokenizer, AutoModelForSeq2SeqLM
```

## Data Loading and cleaning 

```bash
df = pd.read_csv('/content/Approach to Social Media Cyberbullying and Harassment Detection Using Advanced Machine Learning.csv')
df = df.dropna(subset=['Label'])

def clean_text(text):
    text = str(text).lower()
    text = re.sub(r"http\S+|www\S+|https\S+", '', text, flags=re.MULTILINE)
    text = re.sub(r'\@\w+|\#', '', text)
    text = text.translate(str.maketrans('', '', string.punctuation))
    text = re.sub(r'\d+', '', text)
    text = re.sub(r'\s+', ' ', text).strip()
    return text

df['Clean_Text'] = df['Text'].apply(clean_text)
df['Label'] = df['Label'].apply(lambda x: 1 if x.lower() == 'bullying' else 0)
df['text'] = df['Clean_Text']
df = df[['text', 'Label']]
dataset = Dataset.from_pandas(df)
dataset = dataset.train_test_split(test_size=0.2)
```

## Tokenization

```bash
tokenizer = BertTokenizer.from_pretrained("bert-base-uncased")

def tokenize(batch):
    return tokenizer(batch["text"], padding="max_length", truncation=True, max_length=128)

tokenized_dataset = dataset.map(tokenize, batched=True)
tokenized_dataset = tokenized_dataset.rename_column("Label", "labels")
tokenized_dataset.set_format("torch")
```

## Model and Training 

```bash
model = BertForSequenceClassification.from_pretrained("bert-base-uncased", num_labels=2)

training_args = TrainingArguments(
    output_dir="./results",
    eval_strategy="epoch",
    save_strategy="epoch",
    per_device_train_batch_size=8,
    per_device_eval_batch_size=8,
    num_train_epochs=3,
    logging_dir="./logs",
    logging_steps=10,
    load_best_model_at_end=True,
)

trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=tokenized_dataset["train"],
    eval_dataset=tokenized_dataset["test"],
)

trainer.train()
```

## Evaluation

```bash
true_labels = tokenized_dataset["test"]["labels"]
predictions_logits = trainer.predict(tokenized_dataset["test"]).predictions
predictions = np.argmax(predictions_logits, axis=1)

accuracy = accuracy_score(true_labels, predictions)
print(f"Accuracy: {accuracy:.4f}")
print(classification_report(true_labels, predictions))
```

## Toxic-BERT pipeline for classification

```bash
bully_classifier = pipeline(
    "text-classification",
    model="unitary/toxic-bert",
    top_k=3
)
```

## Chatbot Setup

```bash
chat_model_name = "facebook/blenderbot_small-90M"
chat_tokenizer = None
chat_model = None
```

## Custom Gradio UI CSS

```bash
CSS = """
.gradio-container {
    font-family: 'Segoe UI', system-ui, sans-serif;
    background-color: #121a24;
    color: #e0e6ed;
}
... (truncated for brevity)
"""
```

## Detection and Chat Functions

```bash
def detect_bullying(text):
    if not text.strip():
        return "Please enter text to analyze"

    try:
        results = bully_classifier(text)
        output = ["🔍 Analysis Results:"]
        top_labels = []
        bullying_detected = False

        for result in results:
            if isinstance(result, list):
                for r in result:
                    label = r.get('label', 'neutral')
                    score = r.get('score', 0)
                    top_labels.append((label, score))
                    output.append(format_result(label, score))
            else:
                label = result.get('label', 'neutral')
                score = result.get('score', 0)
                top_labels.append((label, score))
                output.append(format_result(label, score))

        bullying_labels = {"toxic", "insult", "hate"}
        for label, score in top_labels:
            if label.lower() in bullying_labels and score > 0.5:
                bullying_detected = True
                break

        if bullying_detected:
            output.insert(1, "🟥 **Classification: Bullying detected**")
        else:
            output.insert(1, "🟩 **Classification: Not Bullying**")

        found_types = detect_specific_phrases(text.lower())
        if found_types:
            output.append("\n🔎 Specific Concerns Detected:")
            output.extend(found_types)

        return "\n".join(output)

    except Exception as e:
        return f"⚠️ Analysis Error: {str(e)}"

def format_result(label, score):
    label_map = {
        'toxic': ("🚨 Toxic Content", "Contains harmful or offensive language"),
        'hate': ("⛔ Hate Speech", "Contains hateful or discriminatory content"),
        'insult': ("⚠️ Insulting Content", "Contains demeaning or insulting language"),
        'neutral': ("✅ Neutral Content", "No clear signs of bullying")
    }
    verdict, explanation = label_map.get(label.lower(),
        ("⚠️ Potentially Harmful", "Content requires review"))

    return (f"{verdict}\n"
            f"Category: {label.title()}\n"
            f"Confidence: {score:.1%}\n"
            f"Note: {explanation}\n"
            f"---")

def detect_specific_phrases(text_lower):
    PHRASE_MAP = {
        'worthless': "This contains demeaning language that attacks self-worth.",
        'disgusting': "This contains degrading or dehumanizing language.",
        'hate': "This appears to contain hateful content.",
        'stupid': "This includes demeaning language targeting intelligence.",
        'ugly': "This appears to contain appearance-based insults.",
        'fuck': "This includes explicit and offensive language."
    }
    return [desc for phrase, desc in PHRASE_MAP.items() if phrase in text_lower]

def mental_health_bot(user_input, chat_history):
    global chat_tokenizer, chat_model
    if not user_input.strip():
        return chat_history

    crisis_words = ["suicide", "kill myself", "end my life", "want to die", "i need help"]
    if any(word in user_input.lower() for word in crisis_words):
        chat_history.append((user_input, f"🚨 I'm very concerned about what you're sharing. {CRISIS_RESOURCES}"))
        return chat_history

    if chat_tokenizer is None or chat_model is None:
        chat_tokenizer = AutoTokenizer.from_pretrained(chat_model_name)
        chat_model = AutoModelForSeq2SeqLM.from_pretrained(chat_model_name)

    inputs = chat_tokenizer([user_input], return_tensors="pt")
    reply_ids = chat_model.generate(**inputs, max_new_tokens=50)
    response = chat_tokenizer.batch_decode(reply_ids, skip_special_tokens=True)[0]

    if len(response.split()) < 5:
        response = "I want to understand better. Can you tell me more about what you're experiencing? If you need immediate help, type 'I need help'."

    chat_history.append((user_input, response))
    return chat_history
```

## Gradio UI Integration

```bash
with gr.Blocks(css=CSS, theme=gr.themes.Soft(primary_hue="blue"), title="SafetyGuard AI") as demo:
    with gr.Column(elem_classes=["header"]):
        gr.Markdown("""# 🛡️ Aura Protect
        *Bullying detection • Mental health support*""")

    with gr.Tabs():
        with gr.Tab("🔍 Scan Content", elem_classes=["tab"]):
            with gr.Row(equal_height=True):
                with gr.Column(min_width=400):
                    input_text = gr.Textbox(
                        label="Text to Analyze",
                        placeholder="Paste content here...",
                        lines=8,
                        max_lines=12,
                        elem_classes=["analysis-box"]
                    )
                    with gr.Row():
                        check_btn = gr.Button("Analyze", variant="primary", scale=2)
                        clear_analysis_btn = gr.Button("Clear", variant="secondary", scale=1)

                with gr.Column(min_width=400):
                    output_text = gr.Textbox(
                        label="Analysis Results",
                        interactive=False,
                        lines=10,
                        show_copy_button=True
                    )
                    gr.Markdown(
                        "<small>⚠️ AI may miss subtle context. Always verify with human judgment.</small>",
                        elem_classes=["crisis-alert"]
                    )

            check_btn.click(detect_bullying, input_text, output_text)
            clear_analysis_btn.click(lambda: ("", ""), None, [input_text, output_text])

        with gr.Tab("💬 Support Chat", elem_classes=["tab"]):
            with gr.Row(equal_height=True):
                with gr.Column(scale=2, min_width=500):
                    chatbot = gr.Chatbot(
                        label="",
                        bubble_full_width=True,
                        height=450,
                        show_label=False
                    )

                with gr.Column(scale=1, min_width=300):
                    msg = gr.Textbox(
                        label="Type your message",
                        placeholder="How can I help?",
                        lines=3,
                        max_lines=5
                    )
                    submit_btn = gr.Button("Send", variant="primary")
                    gr.Markdown("""
                    <div class='crisis-alert' style='margin-top:10px'>
                    <small>💡 Say <strong>"I need help"</strong> for emergency resources</small>
                    </div>
                    """)

            submit_event = msg.submit(
                fn=mental_health_bot,
                inputs=[msg, chatbot],
                outputs=chatbot,
                queue=False
            ).then(lambda: "", None, msg)

            submit_btn.click(
                fn=mental_health_bot,
                inputs=[msg, chatbot],
                outputs=chatbot,
                queue=False
            ).then(lambda: "", None, msg)

    with gr.Accordion("ℹ️ Quick Resources & Info", open=False):
        gr.Markdown(f"""
        **Content Analysis:** Analyzes text using AI (Toxic-BERT model) and keyword matching.
        **Support Chat:** Provides basic emotional support using a conversational AI model (BlenderBot).
        *This tool is not a substitute for professional help.*

        **Crisis Resources:**
        If you're in immediate distress, please contact:
        • National Suicide Prevention Lifeline: 988 (US)
        • Crisis Text Line: Text HOME to 741741 (US)
        • Local emergency services - 100/112

        <small>Version 2.1 | May 2025</small>
        """)

if __name__ == "__main__":
    demo.launch()
```
