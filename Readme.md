# Claude API — Pratik Örnekler

Bu repo, Claude API'nin temel kullanım senaryolarını Jupyter Notebook'lar üzerinden göstermektedir.

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

## Prompt Engineering & Prompt Evaluation

**Prompt Engineering**, modelden istenen çıktıyı tutarlı biçimde elde etmek için komutu tasarlama ve iyileştirme sürecidir; multishot prompting ve XML etiketleriyle yapılandırma gibi teknikler bu kapsamda yer alır. Ancak iyi bir komut yazmak tek başına yeterli değildir — üretim ortamına geçildiğinde kullanıcılar sistemi hiç tahmin edilmeyen yollarla zorlayacak ve geliştirme sırasında sağlam görünen bir komut hızla bozulabilecektir.

**Prompt Evaluation** ise komutun ne kadar güvenilir çalıştığını nesnel olarak ölçer. En sağlam yaklaşım, komutu otomatik bir değerlendirme hattından geçirip geniş test senaryoları üzerinde puanlamak ve bu verilere dayanarak yinelemektir — böylece zayıf noktalar kullanıcıya ulaşmadan yakalanır, farklı komut sürümleri karşılaştırılabilir ve her iyileştirme ölçülebilir kanıta dayanır.
