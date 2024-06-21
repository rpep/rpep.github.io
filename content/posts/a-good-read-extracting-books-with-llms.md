---
title: "Finding 'A Good Read' with Ollama and Llama 3"
date: 2024-06-21T20:42:00+01:00
draft: false
featured_image: '/images/a-good-read.jpg'
---

The BBC has an excellent radio programme called '[A Good Read](https://www.bbc.co.uk/programmes/b006v8jn)'. It first started 1977, and there have been hundreds of episodes, each of which has several celebrities and authors recommending a book each along with the host. Many episodes (though not all, the online archive goes back to roughly the early 1990s) are hosted for free to listen to on the BBC Sounds website. I often like to listen to it, but couldn't find anywhere that the books had been recommended were listed online. Because it's been around a long time, the format of the descriptions isn't consistent and at different points in time, there have been for e.g. more books recommended than at others. I decided to have a try and using Llama locally on my Macbook Pro M2 to try and extract the information. The source code for this post is [here](http://github.com/rpep/a-good-read).


```python
from bs4 import BeautifulSoup
import requests
import ollama
import textwrap
import tqdm
import json
import time
import pandas as pd
from itertools import chain
```

I decided that perhaps the best chance of extracting the information was to pull the long form description of each episode. By doing this, we can extract all the information necessary, but throw away lots of extraneous additional information, reducing the size of the context the LLM would receive. Because the back episodes are hosted on BBC iPlayer, I first had to scrape the actual URLs for each episode of a programme. Each programme has a URL key that describes it, so the following method will scrape the paginated list and return the URLs for each episode:

```python
def find_episodes(programme_id):
    """
    For a programme on BBC iPlayer with a URL:
    https://www.bbc.co.uk/programmes/b006v8jn/episodes/player?page=1
    the ID is "b006v8jn"
    """
    page = 0
    urls_list = []
    while True:
        page += 1
        url = f"https://www.bbc.co.uk/programmes/{programme_id}/episodes/player?page={page}"
        response = requests.get(url)
        if response.status_code == 200:
            html = response.content.decode("UTF-8")
            soup = BeautifulSoup(html, 'html.parser')
            episodes = soup.find_all("div", class_="programme__body", recursive=True)
            if len(episodes) == 0:
                break
            else:
                for episode in episodes:
                    url = episode.h2.a.attrs['href']
                    urls_list.append(url)
        else:
            break
    return urls_list
    
```


```python
urls_list = find_episodes("b006v8jn")
```

Checking that this has worked, we can see that it's pulled the episode list and identified 670 episodes as of the 21st June 2024.

```python
len(urls_list)
```

    670

The next step is to try and grab the descriptions. This proved to be a bit challenging as the pages have changed in content over time, but I was able to identify that on newer pages, the description was contained within a `div` called 'synopsis-toggle__long'. On older pages, it was contained within a div called 'longest-synopsis'. The following code just grabs the description text and strips out superflous newline characters:

```python
texts = []
for url in urls_list:
    response = requests.get(url)
    if response.status_code != 200:
        print("error")
    html = response.content.decode("UTF-8")
    soup = BeautifulSoup(html, 'html.parser')
    text = soup.find("div", class_="synopsis-toggle__long")
    if not text:
        text = soup.find("div", class_="longest-synopsis")
    if not text:
        print(f"skipping url {url}")
        break
    text = text.get_text().replace("\n", " ").lstrip().rstrip()
    texts.append(text)
```

The list of episodes goes from most recent to oldest, so checking that everything worked, we can see that we've got here the episode description for [this](https://www.bbc.co.uk/programmes/m00209gs) episode which had Denise Mina and Simon Brett as guests.

```python
texts[0]
```

```text 
"ABSENT IN THE SPRING by Agatha Christie (writing as Mary Westmacott) (HarperCollins), chosen by Simon BrettIN THE GARDEN OF THE FUGITIVES by Ceridwen Dovey (Penguin), chosen by Denise MinaHIDE MY EYES by Margery Allingham (Penguin), chosen by Harriett Gilbert  Crime writers Denise Mina and Simon Brett join Harriett Gilbert to read each other's favourite books.  Simon Brett (Charles Paris, Fethering and Mrs Pargeter detective series) chooses Agatha Christie under the pseudonym Mary Westmacott, with Absent In The Spring. It’s a story without any detective and one that, perhaps, reveals a more personal side to Christie's writing.  Denise Mina (most recently: Three Fires, The Second Murderer) picks In the Garden of the Fugitives by South African-Australian author Ceridwen Dovey, an epistolary novel which begins with a letter that breaks seventeen years of silence between a rich, elderly man with a broken heart and his former protegee, a young South African filmmaker.  And for the occasion of having two crime authors, Harriett Gilbert picks a golden age crime book, Hide My Eyes by Margery Allingham, where private detective Albert Campion finds himself hunting down a serial killer. Producer: Eliza Lomas for BBC Audio in Bristol Join the conversation @agoodreadbbc Instagram  Show less"
```

The hard part now is trying to extract the books. For this, I provide a prompt to the LLM, and instruct it to extract them, giving an example of the JSON format I would like it to return, using Llama 3's 8B instruct model and the very easy to set up [ollama](https://ollama.com/) to interface with it from Python. The system prompt below went through many iterations to get something that worked reliably, as I found that often superflous fields would appear in the JSON, the titles and authors were badly formatted or missing, authors were confused with the people recommending them, or books weren't found at all. I think that the common refrain I've seen people working with LLMs say of "it's easy to get to 80%, but getting to >95% takes much much longer" certainly matches my experience.

```python
responses = []
for i, text in tqdm.tqdm(enumerate(texts)):
    response = json.loads(
        ollama.chat(
            model='llama3:8b-instruct-q4_0',
            messages=[
                {
                    'role': 'system',
                    'content': textwrap.dedent(
                        """\
                        You are a librarian.
                        You can read text from a description of a radio
                        programme very carefully and extract information about
                        books from it. You should not include information about
                        the people who are on the programme. You should only
                        extract information about the books they have
                        recommended. There are several books to extract from the
                        text, not just one. You only return the author and title
                        of books and no other information. 
                        If the book has been translated, do not include the 
                        translator.
                        
                        Your output should be a list of any detected books in
                        JSON like this:
                        
                        {"books": [
                            {"author": "JRR Tolkien",
                             "title": "Lord of the Rings"},
                            {"author": "JK Rowling",
                             "title": "Harry Potter and the Philosopher's "
                             "Stone"},
                        ]}

                        Fix any grammar issues. For e.g. ensure that everything
                        has the correct capitalisation.
                        """)
                },
                {
                    'role': 'user',
                    'content': text,
                },
            ], 
            format='json')['message']['content'])
    responses.append(response)
```

```text
670it [27:39,  2.48s/it]
```

The execution time for this was, as measured by TQDM, roughly 30 minutes for all 670 text descriptions.

Grabbing the books themselves, we can see that we've found 1986. Older episodes have four books recommended, while newer ones have three, so this sounds roughly right.


```python
books = [r['books'] for r in responses]
books = list(chain.from_iterable(books))
len(books)
```

```text
1986
``````

We can dump the list to a file:


```python
f = open("books.json", 'w')
f.write(json.dumps(books))
```

Briefly inspecting this, we can see that we've mostly accurately pulled the books from the latest few episodes. 

```python
books[:9]
```

```text
[{'author': 'Agatha Christie', 'title': 'Absent In The Spring'},
     {'author': 'Ceridwen Dovey', 'title': 'In the Garden of the Fugitives'},
     {'author': 'Margery Allingham', 'title': 'Hide My Eyes'},
     {'author': 'Barbara Pym', 'title': 'Quartet in Autumn'},
     {'author': 'Rachel Ingalls', 'title': 'Mrs Caliban'},
     {'author': 'Derek Jarman', 'title': 'Pharmacopoeia: A Dungeness Notebook'},
     {'author': 'Julian Barnes', 'title': "Flaubert's Parrot"},
     {'author': 'John Higgs', 'title': 'The KLF'},
     {'author': 'Maggie Nelson', 'title': 'The Red Parts'}]
```


```python
texts[:3]
```


```text
["ABSENT IN THE SPRING by Agatha Christie (writing as Mary Westmacott) (HarperCollins), chosen by Simon BrettIN THE GARDEN OF THE FUGITIVES by Ceridwen Dovey (Penguin), chosen by Denise MinaHIDE MY EYES by Margery Allingham (Penguin), chosen by Harriett Gilbert  Crime writers Denise Mina and Simon Brett join Harriett Gilbert to read each other's favourite books.  Simon Brett (Charles Paris, Fethering and Mrs Pargeter detective series) chooses Agatha Christie under the pseudonym Mary Westmacott, with Absent In The Spring. It’s a story without any detective and one that, perhaps, reveals a more personal side to Christie's writing.  Denise Mina (most recently: Three Fires, The Second Murderer) picks In the Garden of the Fugitives by South African-Australian author Ceridwen Dovey, an epistolary novel which begins with a letter that breaks seventeen years of silence between a rich, elderly man with a broken heart and his former protegee, a young South African filmmaker.  And for the occasion of having two crime authors, Harriett Gilbert picks a golden age crime book, Hide My Eyes by Margery Allingham, where private detective Albert Campion finds himself hunting down a serial killer. Producer: Eliza Lomas for BBC Audio in Bristol Join the conversation @agoodreadbbc Instagram  Show less",
     "QUARTET IN AUTUMN by Barbara Pym, chosen by Samantha Harvey MRS CALIBAN by Rachel Ingalls, chosen by Harriett GilbertPHARMACOPOEIA: A DUNGENESS NOTEBOOK by Derek Jarman, chosen by Darran Anderson Two award-winning writers share books they love with Harriett Gilbert. Samantha Harvey is the author of five novels, The Wilderness, All Is Song, Dear Thief ,The Western Wind and, most recently, Orbital. She is also the author of a memoir, The Shapeless Unease: A Year of Not Sleeping. Her choice of a good read is a slim novel by Barbara Pym set in 1970s London about the lives of four single people in their sixties who work in an office together. Quartet in Autumn is sharply perceptive about the ways in which we hide from one other and from ourselves. Darran Anderson is an Irish writer who lives in London. He is the author of Imaginary Cities: A Tour of Dream Cities, Nightmare Cities, and Everywhere in Between; a memoir, Inventory, about growing up during the Troubles; and the forthcoming In the Land of My Enemy. His choice, Pharmacopoeia, brings together fragments of the artist and filmmaker Derek Jarman's writing on nature, gardening and Prospect Cottage, his Victorian fisherman's hut on the shingle at Dungeness.  Harriett's choice is a fantastically strange novel by Rachel Ingalls, published in 1982. In Mrs Caliban, a grieving housewife in a loveless marriage embarks on a heady affair with a green-skinned frogman.  Produced by Mair Bosworth for BBC Audio  Show less",
     "Historian and author Kathryn Hughes and No Such Thing As a Fish presenter Dan Schreiber recommend favourite books to Harriett Gilbert. Kathryn chooses Flaubert's Parrot by Julian Barnes, an exploration of the French writer's life in the form of a novel. Dan's choice is very different - John Higgs taking on the conceptual artists and chart toppers The KLF. Harriett has gone for Michael Ondaatje's novel Warlight, set in a murky and mysterious post-war London. Presenter: Harriett Gilbert Producer for BBC Audio Bristol: Sally Heaven  Show less"]
```

Analysing the books with Pandas, we can check out whether any were recommended more than once, and what the most popular books were:

```python
df = pd.DataFrame(books)
df
```

<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>author</th>
      <th>title</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>0</th>
      <td>Agatha Christie</td>
      <td>Absent In The Spring</td>
    </tr>
    <tr>
      <th>1</th>
      <td>Ceridwen Dovey</td>
      <td>In the Garden of the Fugitives</td>
    </tr>
    <tr>
      <th>2</th>
      <td>Margery Allingham</td>
      <td>Hide My Eyes</td>
    </tr>
    <tr>
      <th>3</th>
      <td>Barbara Pym</td>
      <td>Quartet in Autumn</td>
    </tr>
    <tr>
      <th>4</th>
      <td>Rachel Ingalls</td>
      <td>Mrs Caliban</td>
    </tr>
    <tr>
      <th>...</th>
      <td>...</td>
      <td>...</td>
    </tr>
    <tr>
      <th>1981</th>
      <td>Nawal El Sadawi</td>
      <td>God Dies on the Nile</td>
    </tr>
    <tr>
      <th>1982</th>
      <td>Dick Francis</td>
      <td>For Kicks</td>
    </tr>
    <tr>
      <th>1983</th>
      <td>Paul Auster</td>
      <td>The New York Trilogy</td>
    </tr>
    <tr>
      <th>1984</th>
      <td>GM Fraser</td>
      <td>Flashman at the Charge</td>
    </tr>
    <tr>
      <th>1985</th>
      <td>Anthony Trollope</td>
      <td>The Small House at Allington</td>
    </tr>
  </tbody>
</table>
<p>1986 rows × 2 columns</p>
</div>

```python
df.value_counts()[:25]
```



```text
    author              title                         
    Angela Carter       Wise Children                     4
    Jean Rhys           Wide Sargasso Sea                 4
    Patrick Hamilton    Hangover Square                   4
    F Scott Fitzgerald  The Great Gatsby                  4
    Chinua Achebe       Things Fall Apart                 3
    Dodie Smith         I Capture The Castle              3
    Evelyn Waugh        A Handful of Dust                 3
    Joan Didion         The Year of Magical Thinking      3
    Sam Selvon          The Lonely Londoners              3
    Robert Graves       Goodbye to All That               3
    Evelyn Waugh        Scoop                             3
    Graham Swift        Waterland                         3
    Bernhard Schlink    The Reader                        3
    Flann O'Brien       The Third Policeman               3
    Graham Greene       Monsignor Quixote                 3
    Muriel Spark        The Girls of Slender Means        3
    Barbara Pym         Excellent Women                   3
    Lorna Sage          Bad Blood                         3
    Annie Proulx        The Shipping News                 3
    Italo Calvino       Invisible Cities                  3
    Michael Frayn       Towards the End of the Morning    3
    Tobias Wolff        Old School                        3
    Kingsley Amis       Lucky Jim                         3
    Jerome K Jerome     Three Men in a Boat               2
    Jane Austen         Pride and Prejudice               2
    Name: count, dtype: int64
```

We can do the same to find the most popular recommended authors:

```python
print(df['author'].value_counts()[:25])
```

```text
    author
    Graham Greene             15
    Evelyn Waugh              14
    Muriel Spark              12
    George Orwell             11
    John Steinbeck            10
    Michael Frayn              9
    Penelope Fitzgerald        9
    Angela Carter              9
    Ian McEwan                 9
    Vladimir Nabokov           8
    Anne Tyler                 8
    Patrick Hamilton           7
    Margaret Atwood            7
    Elizabeth Taylor           7
    Barbara Pym                7
    Philip Roth                7
    Henry James                6
    Gabriel Garcia Marquez     6
    Italo Calvino              6
    Kazuo Ishiguro             6
    Jean Rhys                  6
    Truman Capote              6
    Beryl Bainbridge           6
    PG Wodehouse               6
    Jane Austen                6
    Name: count, dtype: int64
```

While I'm not sure I could ever get through the full list in my lifetime, it's certainly given me a few ideas for books to read outside of my typical sci-fi/fantasy comfort zone! The full JSON dump of extracted books is available [here.](https://github.com/rpep/a-good-read/blob/main/books.json)
