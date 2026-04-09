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
