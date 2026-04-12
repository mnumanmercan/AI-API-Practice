# Claude API — Pratik Örnekler

Bu repo, Claude API'nin temel kullanım senaryolarını Jupyter Notebook'lar üzerinden göstermektedir.

Paylaşılan notlar [Claude With the Anthropic API](https://anthropic.skilljar.com/claude-with-the-anthropic-api) kursundaki dökümanlardan çıkartılmıştır. Daha detaylı bilgi için kursu ziyaret edebilirsiniz.
## System Prompt

`system` parametresi, modelin davranışını ve rolünü tanımlamak için kullanılır. Kullanıcı mesajlarından önce işlenir ve tüm konuşma boyunca geçerliliğini korur. Örneğin [`002_system_prompt.ipynb`](002_system_prompt.ipynb) dosyasında Claude, **sabırlı bir matematik öğretmeni** olarak tanımlanmış ve cevapları doğrudan vermek yerine öğrenciyi adım adım çözüme yönlendirmesi sağlanmıştır:

```python
system = """
    You are a patient math tutor.
    Do not directly answer a student's questions.
    Guide them to a solution step by step.
"""
answer = chat(messages, system=system)
```

## Temperature

`temperature`, modelin yanıt üretirken ne kadar "yaratıcı" ve öngörülemeyen davranacağını belirleyen bir parametredir. `0.0`'a yakın değerler daha tutarlı ve deterministik çıktılar üretirken, `1.0`'a yakın değerler daha özgün ve çeşitli yanıtlar ortaya çıkarır. [`003_temperature.ipynb`](003_temperature.ipynb) dosyasında bu fark, tek cümlelik film fikri üretimi üzerinden somutlaştırılmıştır:

```python
# High temperature - more creative
answer = chat(messages, temperature=1.0)
# → "When a burned-out sleep scientist accidentally discovers that everyone
#    on Earth is sharing the same dream, she has 72 hours to stop an ancient
#    entity from trapping humanity in permanent sleep..."
```

## Response Streaming

Varsayılan API çağrısında model yanıtın tamamını oluşturduktan sonra tek seferde döndürür; bu, uzun yanıtlarda kullanıcının ekranda hiçbir şey görmeden beklediği anlamına gelir. Streaming, yanıtı token token anlık olarak iletir ve kullanıcı deneyimini belirgin biçimde iyileştirir — ChatGPT benzeri uygulamalardaki "yazıyormuş gibi" hissi tam olarak bu yapıyla sağlanır. [`004_response_stream.ipynb`](004_response_stream.ipynb) dosyasında iki yöntem gösterilmiştir: `stream=True` ile ham event'lere erişim ve `client.messages.stream()` ile daha sade bir text stream:

```python
# Ham event akışı — her token bir RawContentBlockDeltaEvent olarak gelir
stream = client.messages.create(model=model, max_tokens=1000, messages=messages, stream=True)
for event in stream:
    print(event)

# Sade text stream — yalnızca metin parçaları, karakter karakter yazdırılır
with client.messages.stream(model=model, max_tokens=1000, messages=messages) as stream:
    for text in stream.text_stream:
        print(text, end="")
```

## Prompt Engineering Techniques

**Prompt Engineering**, modelden tutarlı ve yapılandırılmış çıktılar elde etmek için komutun nasıl tasarlandığına dair bir dizi pratik tekniği kapsar. [`001_prompting.ipynb`](Prompt-Eng-Techniques/001_prompting.ipynb) dosyasında bu tekniklerin birlikte nasıl çalıştığı somut bir pipeline üzerinden gösterilmektedir: **Prefill + Stop Sequences** ile model, yanıtını doğrudan `json` bloğundan başlatmaya ve kapanış backtick'ini gördüğünde durmaya zorlanır; bu sayede `json.loads()` için ek temizleme adımı gerekmez. **XML etiketleri** (`<task_description>`, `<criteria>`, `<solution>` vb.) ise bağlamı açık sınırlarla bölümlere ayırarak modelin hangi parçayı nasıl yorumlaması gerektiğini netleştirir ve tutarlı çıktı üretimi sağlar.

Teknikler yalnızca tek bir noktada değil, birbirini tamamlayacak biçimde katmanlanır. **Template tabanlı prompting** (`render()` fonksiyonu), prompt metninin içindeki `{değişken}` yer tutucularını çalışma zamanında değiştirerek aynı prompt şablonunu farklı senaryolara uyarlamayı kolaylaştırır. **Few-shot (çok örnekli) prompting**, `<sample_input>` / `<ideal_output>` blokları halinde modele beklenen çıktı biçimini doğrudan öğretirken, **sıcaklık kontrolü** (`temperature=1.0` fikir üretiminde, `temperature=0.0` puanlamada) görevin yaratıcılık-tutarlılık dengesine göre ayarlanır. Tüm bu teknikler bir arada kullanıldığında prompt hem esnekliğini hem güvenilirliğini korur.

```python
# Prefill + Stop Sequences — model JSON'dan başlar, kapanış ``` işaretinde durur
messages = []
add_user_message(messages, rendered_prompt)
add_assistant_message(messages, "```json")          # prefill: yanıt buradan başlar
text = chat(messages, stop_sequences=["```"])       # kapanış backtick'inde dur
result = json.loads(text)                           # doğrudan parse edilebilir çıktı

# XML etiketleri + Few-shot örnek — bağlam bölümlere ayrılır, beklenen format gösterilir
prompt = """
<task_description>
{task}
</task_description>

<sample_input>
  <content>Güneş enerjisi maliyetleri 2010'dan bu yana %89 düştü...</content>
</sample_input>
<ideal_output>
```json
{"solution_criteria": ["Tüm konular listelenmeli"]}
```
</ideal_output>

Şimdi şu girdi için aynı formatı üret:
<content>{actual_content}</content>
"""

# Template render — aynı şablon farklı senaryolara uyarlanır
rendered = evaluator.render(prompt, {"task": task_desc, "actual_content": content})

# Sıcaklık kontrolü — fikir üretiminde yaratıcılık, puanlamada deterministiklik
ideas  = chat(messages, temperature=1.0)   # çeşitli ve özgün fikirler
grades = chat(messages, temperature=0.0)   # tekrarlanabilir, kararlı puanlama
```

## Prompt Engineering & Prompt Evaluation

**Prompt Engineering**, modelden istenen çıktıyı tutarlı biçimde elde etmek için komutu tasarlama ve iyileştirme sürecidir; multishot prompting ve XML etiketleriyle yapılandırma gibi teknikler bu kapsamda yer alır. Ancak iyi bir komut yazmak tek başına yeterli değildir — üretim ortamına geçildiğinde kullanıcılar sistemi hiç tahmin edilmeyen yollarla zorlayacak ve geliştirme sırasında sağlam görünen bir komut hızla bozulabilecektir.

**Prompt Evaluation** ise komutun ne kadar güvenilir çalıştığını nesnel olarak ölçer. En sağlam yaklaşım, komutu otomatik bir değerlendirme hattından geçirip geniş test senaryoları üzerinde puanlamak ve bu verilere dayanarak yinelemektir — böylece zayıf noktalar kullanıcıya ulaşmadan yakalanır, farklı komut sürümleri karşılaştırılabilir ve her iyileştirme ölçülebilir kanıta dayanır.

## Prompt Evals — Adım Adım Değerlendirme Hattı

Bir eval pipeline'ının ilk adımı güvenilir bir test dataseti oluşturmaktır. [`006_prompt_evals.ipynb`](006_prompt_evals.ipynb) dosyasında bu dataset yine Claude'un kendisiyle üretilmiştir — modelden AWS ile ilgili Python, JSON veya Regex görevleri içeren bir JSON dizisi oluşturması istenmiş, yanıtın doğrudan geçerli JSON olarak başlaması için `assistant` prefill ve `stop_sequences` birlikte kullanılmıştır. Dataset [`dataset.json`](dataset.json) dosyasına kaydedilir; her nesne tek bir görev tanımı içerir ve tüm eval adımlarına girdi sağlar:

```python
add_user_message(messages, prompt)
add_assistant_message(messages, "```json")      # prefill — model JSON'dan başlar
text = chat(messages, stop_sequences=["```"])   # kapanış backtick'inde dur
return json.loads(text)
```

İkinci adımda [`007_prompt_evals_grader.ipynb`](007_prompt_evals_grader.ipynb) ile **model tabanlı puanlama** devreye girer. Claude'un ürettiği her çıktı, yine Claude'a bir reviewer olarak gönderilir; model çıktıyı `strengths`, `weaknesses`, `reasoning` ve `score` (1–10) alanlarından oluşan yapılandırılmış bir JSON ile değerlendirir. Bu yaklaşımda da prefill + stop_sequences kullanılarak grader'ın çıktısı doğrudan `json.loads()` ile ayrıştırılır. Sonuçta elde edilen ortalama skor **7.17**'dir:

```python
def grade_by_model(test_case, output):
    # eval_prompt içinde <task> ve <solution> XML etiketleriyle bağlam sağlanır
    add_assistant_message(messages, "```json")
    eval_text = chat(messages, stop_sequences=["```"])
    return json.loads(eval_text)   # → {"strengths": [...], "score": 8, ...}
```

Üçüncü adımda [`008_prompt_evals_fns.ipynb`](008_prompt_evals_fns.ipynb) ile **syntax doğrulama** eklenir. Model puanlamasına ek olarak çıktının gerçekten çalışabilir kod olup olmadığı `ast.parse()`, `json.loads()` ve `re.compile()` ile denetlenir; sözdizimi geçerliyse 10, değilse 0 puan verilir. Nihai skor bu iki puanın ortalamasıdır. Aynı zamanda prompt da sıkılaştırılmış — `run_prompt()` fonksiyonu artık modeli yalnızca ham kod üretmeye yönlendiriyor ve çıktısını `"```code"` prefill ile temizliyor. Bu değişiklikler ortalama skoru **8.0**'a taşır:

```python
syntax_score = grade_syntax(output, test_case)   # 10 ya da 0
score = (model_score + syntax_score) / 2         # kombine skor
```

Son adımda [`009_prompt_evals_complete.ipynb`](009_prompt_evals_complete.ipynb) ile dataset şemasına `solution_criteria` alanı eklenir. Grader artık göreve özel kriterler üzerinden değerlendirir; bu sayede puanlama daha tutarlı ve nesnel hale gelir. Dört aşamanın birleşik sonucu olarak ortalama skor **8.17**'ye ulaşır. Tüm pipeline şu döngüyü izler: dataset oluştur → her görevi modele çözdür → hem model hem syntax ile puanla → ortalama skoru raporla:

```
generate_dataset() → run_prompt() → grade_by_model() + grade_syntax() → run_eval()
Avg score: 7.17  →  8.0  →  8.17   (her adımda ölçülebilir iyileşme)
```
