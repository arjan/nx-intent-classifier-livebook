<!-- livebook:{"persist_outputs":true} -->

# Building an intent classifier using Nx, Axon and BERT

## Setup

<!-- livebook:{"reevaluate_automatically":true} -->

```elixir
Mix.install([
  {:req, "~> 0.2.0"},
  {:axon, "~> 0.1.0-dev", github: "elixir-nx/axon", branch: "main"},
  {:exla, "~> 0.1.0-dev", github: "elixir-nx/nx", sparse: "exla"},
  {:nx, "~> 0.1.0-dev", github: "elixir-nx/nx", sparse: "nx", override: true}
])
```

```output
* Getting axon (https://github.com/elixir-nx/axon.git - origin/main)
remote: Enumerating objects: 2646, done.        
remote: Counting objects: 100% (1822/1822), done.        
remote: Compressing objects: 100% (1077/1077), done.        
remote: Total 2646 (delta 1351), reused 1120 (delta 726), pack-reused 824        
* Getting exla (https://github.com/elixir-nx/nx.git)
remote: Enumerating objects: 10273, done.        
remote: Counting objects: 100% (1891/1891), done.        
remote: Compressing objects: 100% (197/197), done.        
remote: Total 10273 (delta 1775), reused 1713 (delta 1691), pack-reused 8382        
origin/HEAD set to main
* Getting nx (https://github.com/elixir-nx/nx.git)
remote: Enumerating objects: 10273, done.        
remote: Counting objects: 100% (1945/1945), done.        
remote: Compressing objects: 100% (194/194), done.        
remote: Total 10273 (delta 1831), reused 1770 (delta 1748), pack-reused 8328        
origin/HEAD set to main
Resolving Hex dependencies...
Dependency resolution completed:
New:
  castore 0.1.13
  elixir_make 0.6.3
  finch 0.8.3
  jason 1.2.2
  mime 2.0.2
  mint 1.4.0
  nimble_options 0.3.7
  nimble_pool 0.2.4
  req 0.2.0
  table_rex 3.1.1
  telemetry 1.0.0
  xla 0.2.0
* Getting req (Hex package)
* Getting xla (Hex package)
* Getting elixir_make (Hex package)
* Getting table_rex (Hex package)
* Getting finch (Hex package)
* Getting jason (Hex package)
* Getting mime (Hex package)
* Getting castore (Hex package)
* Getting mint (Hex package)
* Getting nimble_options (Hex package)
* Getting nimble_pool (Hex package)
* Getting telemetry (Hex package)
==> nimble_options
Compiling 3 files (.ex)
Generated nimble_options app
==> nimble_pool
Compiling 2 files (.ex)
Generated nimble_pool app
==> nx
Compiling 20 files (.ex)
Generated nx app
===> Analyzing applications...
===> Compiling telemetry
==> jason
Compiling 8 files (.ex)
Generated jason app
==> elixir_make
Compiling 1 file (.ex)
Generated elixir_make app
==> xla
Compiling 2 files (.ex)
Generated xla app

13:30:43.740 [info]  Found a matching archive (xla_extension-x86_64-linux-cpu.tar.gz), going to download it

13:30:52.184 [info]  Successfully downloaded the XLA archive
==> exla
Unpacking /home/arjan/.cache/xla/0.2.0/cache/download/xla_extension-x86_64-linux-cpu.tar.gz into /home/arjan/.cache/mix/installs/elixir-1.12.2-erts-12.1.3/a04fb5c0bd902d7099cad6291c1b8daa/deps/exla/exla/cache
mkdir -p /home/arjan/.cache/mix/installs/elixir-1.12.2-erts-12.1.3/a04fb5c0bd902d7099cad6291c1b8daa/_build/dev/lib/exla/priv
ln -sf /home/arjan/.cache/mix/installs/elixir-1.12.2-erts-12.1.3/a04fb5c0bd902d7099cad6291c1b8daa/deps/exla/exla/cache/xla_extension/lib /home/arjan/.cache/mix/installs/elixir-1.12.2-erts-12.1.3/a04fb5c0bd902d7099cad6291c1b8daa/_build/dev/lib/exla/priv/lib
g++ -fPIC -I/home/arjan/.asdf/installs/erlang/24.1.3/erts-12.1.3/include -isystem cache/xla_extension/include -O3 -Wall -Wextra -Wno-unused-parameter -Wno-missing-field-initializers -Wno-comment -shared -std=c++14 c_src/exla/exla.cc c_src/exla/exla_nif_util.cc c_src/exla/exla_client.cc -o /home/arjan/.cache/mix/installs/elixir-1.12.2-erts-12.1.3/a04fb5c0bd902d7099cad6291c1b8daa/_build/dev/lib/exla/priv/libexla.so -L/home/arjan/.cache/mix/installs/elixir-1.12.2-erts-12.1.3/a04fb5c0bd902d7099cad6291c1b8daa/_build/dev/lib/exla/priv/lib -lxla_extension -Wl,-rpath,'$ORIGIN/lib'
c_src/exla/exla_client.cc: In function ‘tensorflow::StatusOr<std::vector<exla::ExlaBuffer*> > exla::UnpackRunArguments(ErlNifEnv*, ERL_NIF_TERM, exla::ExlaClient*, int)’:
c_src/exla/exla_client.cc:89:19: warning: redundant move in return statement [-Wredundant-move]
   89 |   return std::move(arg_buffers);
      |          ~~~~~~~~~^~~~~~~~~~~~~
c_src/exla/exla_client.cc:89:19: note: remove ‘std::move’ call
Compiling 15 files (.ex)
Generated exla app
==> castore
Compiling 1 file (.ex)
Generated castore app
==> mint
Compiling 1 file (.erl)
Compiling 23 files (.ex)
Generated mint app
==> finch
Compiling 11 files (.ex)
Generated finch app
==> mime
Compiling 1 file (.ex)
Generated mime app
==> req
Compiling 4 files (.ex)
Generated req app
==> table_rex
Compiling 7 files (.ex)
Generated table_rex app
==> axon
Compiling 20 files (.ex)
Generated axon app
```

```output
:ok
```

## The embeddings server

The embeddings server is a python program that runs on the following URL:

```elixir
server = "http://localhost:5000/encode"
```

```output
"http://localhost:5000/encode"
```

The Python repo for it is [located on Github](https://github.com/arjan/embeddings-server-ref). 
Lets try out the embedding server:

```elixir
bytes = Req.get!("#{server}?q=Hello+world").body
```

```output
<<4, 164, 2, 60, 60, 206, 12, 62, 52, 71, 35, 188, 56, 32, 141, 61, 65, 170, 88, 62, 116, 130, 127,
  60, 147, 90, 220, 187, 159, 191, 44, 61, 156, 225, 132, 61, 70, 245, 98, 60, 17, 89, 91, 61, 227,
  48, 211, 189, 168, 246, ...>>
```

Lets put this data into a Tensor

```elixir
bytes |> Nx.from_binary({:f, 32})
```

```output
#Nx.Tensor<
  f32[768]
  [0.007973674684762955, 0.13750547170639038, -0.009965706616640091, 0.06890910863876343, 0.21158696711063385, 0.015595067292451859, -0.0067246644757688046, 0.04217493161559105, 0.06488344073295593, 0.013852423056960106, 0.05355173721909523, -0.1031205877661705, -0.04076257348060608, 0.031445689499378204, 0.23194989562034607, -0.28180691599845886, 0.13213446736335754, -0.06846168637275696, -0.33199381828308105, -0.04467453062534332, 0.10618312656879425, -0.03570239618420601, 0.10691921412944794, -0.0746450275182724, 0.07432293146848679, 0.09699782729148865, -0.08254548162221909, 0.14649392664432526, 0.07160726189613342, 0.06751589477062225, 0.10908330976963043, 0.021265951916575432, 0.0757104679942131, -0.029477950185537338, 0.12306474149227142, -0.0727241113781929, 0.0667380690574646, -0.04435543343424797, -0.13669823110103607, -0.03493381291627884, 0.10793402791023254, -0.07988861948251724, -0.11097834259271622, 0.11092036962509155, -0.17504799365997314, -0.11487087607383728, -0.04099629819393158, -0.10665401071310043, -0.006150360684841871, 0.10163969546556473, ...]
>
```

```elixir
dim = bytes |> Nx.from_binary({:f, 32}) |> Nx.size()
```

```output
768
```

Okay, so that works, we can make a tensor out of a sentence. In the next part we are going to extend this to translate a list of annotated sentences into an array of vectors with their corresponding labels.

## Loading the training data

Now, we are going to translate a list of annotated sentences into an array of vectors with their corresponding labels.
This is the input file:

```elixir
json_data = "disco.json" |> File.read!() |> Jason.decode!()
```

```output
[
  %{
    "id" => "faq_about",
    "training_data" => ["tell me something about yourself", "what are you",
     "i want to know everything about you", "i want to know about you", "why are you here",
     "I want to know you better", "describe yourself", "talk about yourself",
     "explain what you are", "what is this exactly", "are you a chatbot",
     "i want to know more about you", "can you introduce yourself", "please introduce yourself",
     "tell me about yourself"]
  },
  %{
    "id" => "faq_purpose",
    "training_data" => ["what is the purpose", "i want to know what your purpose is",
     "what is the idea?", "what do you need from me?", "what is the reason for your existence?",
     "what do you want from me?", "can you tell me what your purpose is?",
     "what is the purpose exactly?", "what is your purpose?", "what is the idea with this"]
  },
  %{
    "id" => "faq_about_us",
    "training_data" => ["more details on this project", "i have a question about the company",
     "about the project", "tell me about your company", "can you tell some more about you guys",
     "what do you guys do", "give me more information about your game",
     "i'm looking for more information about your company", "who made this project",
     "who made this game", "who are the builders of this project",
     "who are the creators of this game"]
  },
  %{
    "id" => "faq_address",
    "training_data" => ["where are you based?", "what is the location of your company?",
     "can you tell me the address of your company?", "can you tell me the address?",
     "what's your address exactly?", "where are you?", "in which country are you guys based?",
     "where are you guys located?", "on what address is your office?",
     "on what street is your office?", "what is your office address?", "where is your office?",
     "where are you located?", "what is your address?"]
  },
  %{
    "id" => "faq_contactdetails",
    "training_data" => ["can you share with me your contact details", "i want contact details",
     "i want your contact information", "how can i get in touch with you guys",
     "how can i get in touch with your company", "how can i reach you",
     "do you have information about how to reach you", "what are your contact details",
     "I'm looking for your contact details", "give me your contactdetails",
     "give me your contact details"]
  },
  %{
    "id" => "faq_email",
    "training_data" => ["i would like to mail",
     "I would like to mail to the sales department, do you have an email",
     "how can i reach you by email?", "what is your email address?", "tell me your email",
     "do you have an email address?", "please give me your email", "your email please",
     "where can i send my emails to?", "how can i find email address?",
     "to which address can i send an email?", "what is your email?"]
  },
  %{
    "id" => "faq_phone",
    "training_data" => ["can i call the service desk?", "phone number", "phonenumber",
     "what is the phone number of the customer service department?", "can i give you a call",
     "i want to call the helpdesk", "what is phone number of customer services",
     "what is your phone number?", "i want to give you a call", "can i call you up?",
     "how can i call you?"]
  },
  %{
    "id" => "faq_what_do_i_need",
    "training_data" => ["What do I need to join a distance disco?", "What do I need to join?",
     "How can I join?", "What do I need"]
  },
  %{
    "id" => "faq_how_does_it_work",
    "training_data" => ["What am I supposed to do?", "How does this work?",
     "How does the disco work?", "how does it work"]
  },
  %{
    "id" => "faq_webcam_does_not_work",
    "training_data" => ["My webcam does not work?", "My laptop camera is not working",
     "My webcam image stays black", "i have no camera", "my camera stays black",
     "browser does not ask for camera permission"]
  },
  %{
    "id" => "faq_black_screen",
    "training_data" => ["I only see black screens?", "I do not see any video frames"]
  },
  %{
    "id" => "faq_mobile",
    "training_data" => ["will it work on ipad", "can I use my tablet?", "can I use my android?",
     "does it work on an android tablet?", "Can I use my phone or tablet?",
     "does it work on an iphone?", "i want to view it on an ipad", "i want it on an iphone"]
  },
  %{
    "id" => "faq_freeze",
    "training_data" => ["The music stopped / everything seems to freeze?", "My computer froze",
     "I do not see any video anymore", "Everybody is frozen"]
  },
  %{
    "id" => "faq_mic_is_off",
    "training_data" => ["How do I switch on my mic?", "I want to talk during the disco",
     "can you talk during the disco", "the sound is off", "I cannot hear people",
     "can I talk to other people while dancing?"]
  },
  %{
    "id" => "faq_headphones",
    "training_data" => ["Do I need headphones?", "should I use headphones"]
  },
  %{
    "id" => "faq_chat",
    "training_data" => ["Can I chat during the disco?", "I want to chat while we are dancing "]
  },
  %{
    "id" => "faq_grid_view",
    "training_data" => ["I only see one person very big?", "I see one person very large",
     "there is one person zoomed in"]
  },
  %{
    "id" => "faq_zoom_in",
    "training_data" => ["How can I zoom in on one person?", "Can I view just one person?",
     "I want to view only a single person"]
  },
  %{
    "id" => "faq_skip_song",
    "training_data" => ["I hate this song", "What if I don't like a song",
     "I want to go to the next song", "next song please", "skip this song"]
  },
  %{
    "id" => "faq_pause",
    "training_data" => ["I want to take a break to get some fresh drinks.",
     "I want to take a break", "I need to rest for a while"]
  },
  %{
    "id" => "faq_own_music",
    "training_data" => ["i want to listen to my own music",
     "Can I use my own music with Distance Disco?", "i want my own playlist",
     "can I play a customized playlist?", "can I connect spotify account?"]
  },
  %{
    "id" => "faq_winning",
    "training_data" => ["is there a prize to win?", "what can I win?", "how can I win this game?",
     "i want a prize", "I think I have won"]
  },
  %{
    "id" => "faq_howmany",
    "training_data" => ["How many people can join?", "With how many persons can I participate?",
     "How many folks can join together in the disco?", "How many persons?",
     "Is there a maximum capacity for a disco?", "What is the best size for a disco?",
     "How big can a disco become?", "How many people can play the game?"]
  },
  %{
    "id" => "faq_donate",
    "training_data" => ["Can I still donate to your project?", "I want to give you some money",
     "I want to donate", "I want to donate some money"]
  },
  %{
    "id" => "faq_price",
    "training_data" => ["How much does it cost?", "How much do I have to pay?", "Do I need money??",
     "How much money does it cost?", "What is the price for a distance disco?",
     "What is the price?", "Is the price to make a disco per person or total?",
     "Do participants need to pay?", "How much does everyone needs to pay?"]
  },
  %{
    "id" => "faq_try_out",
    "training_data" => ["Can I first try out the disco before buying?", "Is there a trial version?",
     "Can I test?", "can I test a disco before I buy it", "is there a way to try it out first"]
  },
  %{
    "id" => "faq_change_time",
    "training_data" => ["Is it possible to change the time of the disco?",
     "I want to change the start time of the disco", "I want to postpone the disco",
     "can I make the disco start sooner", "i want the disco to start earlier",
     "can I make the disco start later? "]
  },
  %{
    "id" => "faq_email_span",
    "training_data" => ["i didn't receive an email after creating a disco",
     "I don't get email messages", "E-mail does not arrive", "I did not get the confirmation mail"]
  },
  %{
    "id" => "faq_playlist_option",
    "training_data" => ["Do you have music for kids?", "Do you have music for children?",
     "Do you have a playlist for kids?", "Do you have a playlist for children?",
     "Is there a special version for children", "Do you have a carnaval playlist?",
     "What kind of music do you have?", "What kind of songs can I expect?", "Can I create a theme?",
     ""]
  },
  %{
    "id" => "faq_when_does_it_start",
    "training_data" => ["When does the disco start?", "Can I join the disco before it starts?",
     "Is there a waiting room for people that come too early?", "When does the disco open?"]
  },
  %{
    "id" => "fallback",
    "training_data" => ["forklar hvad du er", "explain what you are", "Rede über dich selbst",
     "erkläre was du bist", "habla de ti mismo", "Kuvaile itseäsi", "décrivez-vous",
     "Ik wil alles van je weten", "Jeg vil gerne eskalere til et menneske",
     "jeg sætter pris på det", "Danke, Mann", "ja, ich bin sicher", "das ist der, den ich meine",
     "can we do that one more time please?", "can i try again please", "nothing", "no don't stop",
     ...]
  }
]
```

Lets transform it into a list of labels and a list of sentences:

```elixir
{labels, utterances} =
  json_data
  |> Enum.flat_map(fn %{"id" => id, "training_data" => training_data} ->
    training_data |> Enum.map(&{id, &1})
  end)
  |> Enum.unzip()
```

```output
{["faq_about", "faq_about", "faq_about", "faq_about", "faq_about", "faq_about", "faq_about",
  "faq_about", "faq_about", "faq_about", "faq_about", "faq_about", "faq_about", "faq_about",
  "faq_about", "faq_purpose", "faq_purpose", "faq_purpose", "faq_purpose", "faq_purpose",
  "faq_purpose", "faq_purpose", "faq_purpose", "faq_purpose", "faq_purpose", "faq_about_us",
  "faq_about_us", "faq_about_us", "faq_about_us", "faq_about_us", "faq_about_us", "faq_about_us",
  "faq_about_us", "faq_about_us", "faq_about_us", "faq_about_us", "faq_about_us", "faq_address",
  "faq_address", "faq_address", "faq_address", "faq_address", "faq_address", "faq_address",
  "faq_address", "faq_address", "faq_address", "faq_address", "faq_address", ...],
 ["tell me something about yourself", "what are you", "i want to know everything about you",
  "i want to know about you", "why are you here", "I want to know you better", "describe yourself",
  "talk about yourself", "explain what you are", "what is this exactly", "are you a chatbot",
  "i want to know more about you", "can you introduce yourself", "please introduce yourself",
  "tell me about yourself", "what is the purpose", "i want to know what your purpose is",
  "what is the idea?", "what do you need from me?", "what is the reason for your existence?",
  "what do you want from me?", "can you tell me what your purpose is?",
  "what is the purpose exactly?", "what is your purpose?", "what is the idea with this",
  "more details on this project", "i have a question about the company", "about the project",
  "tell me about your company", "can you tell some more about you guys", "what do you guys do",
  "give me more information about your game", "i'm looking for more information about your company",
  "who made this project", "who made this game", "who are the builders of this project",
  "who are the creators of this game", "where are you based?",
  "what is the location of your company?", "can you tell me the address of your company?",
  "can you tell me the address?", "what's your address exactly?", "where are you?",
  "in which country are you guys based?", "where are you guys located?",
  "on what address is your office?", "on what street is your office?",
  "what is your office address?", ...]}
```

Let's also save the number of samples:

```elixir
n_samples = Enum.count(utterances)
```

```output
227
```

And the number of labels:

```elixir
n_labels = labels |> Enum.uniq() |> Enum.count()
```

```output
31
```

## Encoding the training data

First of all, the labels. We want to encode these as numbers instead of text labels, 
and make them into a [1-hot encoded](https://en.wikipedia.org/wiki/One-hot) tensor:

```elixir
lookup = labels |> Enum.uniq() |> Enum.with_index() |> Map.new()
reverse_lookup = lookup |> Enum.map(fn {k, v} -> {v, k} end) |> Map.new()

train_labels =
  labels
  |> Enum.map(&lookup[&1])
  |> Nx.tensor()
  |> Nx.new_axis(-1)
  |> Nx.equal(Nx.tensor(Enum.to_list(0..(n_labels - 1))))
```

```output
#Nx.Tensor<
  u8[227][31]
  [
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0],
    [1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, ...],
    ...
  ]
>
```

Now, lets convert the utterances to a tensor. We do this the same way as before, but now with a POST method to the embeddings server because we are encoding many sentences in one request:

```elixir
body = utterances |> Enum.map(&{:q, &1}) |> URI.encode_query()

headers = %{"content-type" => "application/x-www-form-urlencoded"}
data = Req.post!(server, body, headers: headers).body
```

```output
<<64, 185, 238, 189, 121, 223, 51, 62, 157, 162, 85, 188, 242, 235, 138, 188, 247, 207, 46, 61, 70,
  127, 203, 61, 79, 202, 14, 190, 185, 225, 24, 190, 167, 85, 228, 61, 88, 108, 22, 61, 129, 125,
  176, 61, 101, 123, 212, 61, 192, 179, ...>>
```

... and make it into a tensor again, shaped accordingly:

```elixir
train_data = Nx.from_binary(data, {:f, 32}) |> Nx.reshape({n_samples, dim})
```

```output
#Nx.Tensor<
  f32[227][768]
  [
    [-0.11656427383422852, 0.17565716803073883, -0.013039258308708668, -0.01695821061730385, 0.042678799480199814, 0.09936384856700897, -0.13944362103939056, -0.1492985635995865, 0.11149149388074875, 0.03672441840171814, 0.08617687970399857, 0.10375098139047623, -0.06284284591674805, 0.12252988666296005, 0.10226114094257355, -0.1914869248867035, 0.00964775588363409, -0.00845492072403431, 0.1347908228635788, 0.014932033605873585, -0.16111542284488678, -0.04643867164850235, 0.09877096116542816, -0.1250312775373459, -0.0940314307808876, -0.013486901298165321, 0.11868830770254135, -0.018658753484487534, 0.10667247325181961, 0.037525273859500885, 0.07248277217149734, -0.19075097143650055, -0.05292833596467972, -0.15352872014045715, 0.07912596315145493, -0.03185955807566643, 0.032129500061273575, 0.02027236297726631, -0.06591279804706573, -0.19240720570087433, 0.16795043647289276, 0.1802370250225067, 0.17723263800144196, 0.127309650182724, -0.12330614030361176, -0.18257081508636475, 0.1026843935251236, 0.029442543163895607, 0.05434111878275871, 0.12262914329767227, ...],
    ...
  ]
>
```

## Training the classifier

Alright! now the fun stuff can start. let's use *Axon* to train a classifier. I use a pretty
standard neural net here with *softmax* activation. You know what they say about AI:

![XKCD AI](https://imgs.xkcd.com/comics/machine_learning.png)

But for reference, I adapted the model below from the 
[Axon MNIST example](https://github.com/elixir-nx/axon/blob/main/examples/mnist.exs):

```elixir
model =
  Axon.input({nil, dim})
  |> Axon.dense(128, activation: :relu)
  |> Axon.dense(n_labels, activation: :softmax)
```

```output
---------------------------------------------------
                       Model
===================================================
 Layer                     Shape        Parameters
===================================================
 input_97 ( input )        {nil, 768}   0
 dense_100 ( dense )       {nil, 128}   98432
 relu_101 ( relu )         {nil, 128}   0
 dense_104 ( dense )       {nil, 31}    3999
 softmax_105 ( softmax )   {nil, 31}    0
---------------------------------------------------

```

And let's train it

```elixir
epochs = 10
bs = 30

data =
  Stream.zip(
    Nx.to_batched_list(train_data, bs),
    Nx.to_batched_list(train_labels, bs)
  )

model_state =
  model
  |> Axon.Loop.trainer(:categorical_cross_entropy, Axon.Optimizers.adamw(0.005))
  |> Axon.Loop.metric(:accuracy, "Accuracy")
  |> Axon.Loop.run(data, epochs: epochs, compiler: EXLA)
```

```output
%{
  "dense_100" => %{
    "bias" => #Nx.Tensor<
      f32[128]
      [0.043292734771966934, 0.014375993981957436, 0.0786755308508873, -0.00985157210379839, 0.0114497235044837, 0.06380867958068848, 0.060464587062597275, 0.05320773646235466, 0.02041640132665634, -0.009514473378658295, 0.026460770517587662, 0.04908758029341698, 0.011929018422961235, -0.03286842629313469, 0.04439902678132057, 0.03158632665872574, -0.007127372547984123, 0.01014319434762001, 0.022989151999354362, 0.03059743531048298, 0.08820179104804993, 0.05882834643125534, 0.0263373926281929, 0.04388979822397232, 0.042668990790843964, 0.038027841597795486, 0.05875687301158905, 0.04237501695752144, 0.07104531675577164, -0.04072129353880882, 0.05334039777517319, 0.043321020901203156, 0.024670470505952835, 0.07207701355218887, 0.057019565254449844, 0.03844977915287018, 0.029690410941839218, 0.05302223563194275, 0.01346414815634489, 0.016728444024920464, 0.018509449437260628, -8.150069043040276e-4, -0.018211448565125465, 0.014235900714993477, 0.0383852981030941, -0.02480454556643963, 0.02712310664355755, 0.06064534932374954, ...]
>,
    "kernel" => #Nx.Tensor<
      f32[768][128]
      [
        [-0.016703404486179352, -0.13331551849842072, 0.18462680280208588, 0.10387539118528366, -0.06920168548822403, 0.0037816346157342196, -0.05017930641770363, -0.20827548205852509, -0.029789665713906288, -0.027188312262296677, -0.030752448365092278, -0.05519084259867668, -0.05347512662410736, 0.051177091896533966, 0.04407903924584389, 0.04689871892333031, -0.09903385490179062, 0.004762102384120226, -0.12395315617322922, -0.09948376566171646, 0.010023055598139763, 0.058849673718214035, -0.07826945185661316, -0.05101356655359268, -0.0038907884154468775, 0.053499069064855576, 0.06305880099534988, -0.12270945310592651, 0.012656410224735737, -0.01626901514828205, 0.09370135515928268, 0.1418253481388092, -0.004680234473198652, 0.14784762263298035, -0.13576242327690125, 0.09420953691005707, 0.07475412636995316, -0.05164937674999237, 0.10531721264123917, -0.05053820461034775, -0.005199302453547716, -0.0879281535744667, 0.08944553881883621, 0.05082683637738228, -0.026636414229869843, 0.06862376630306244, -0.10023453086614609, ...],
        ...
      ]
>
  },
  "dense_104" => %{
    "bias" => #Nx.Tensor<
      f32[31]
      [-0.01599293202161789, -0.017059624195098877, -0.017199015244841576, -0.017791494727134705, -0.019637027755379677, -0.00868568941950798, 0.00856764055788517, -0.03506317734718323, -0.06669768691062927, -0.010809218510985374, -0.06910909712314606, -0.007028351537883282, -0.015895383432507515, -0.013899238780140877, -0.07168107479810715, -0.06486164033412933, -0.051182590425014496, -0.032707329839468, -0.0038463796954602003, -0.030523747205734253, -0.020893752574920654, -0.01229503471404314, -0.0190647654235363, -0.006014500744640827, -0.016089508309960365, -0.001700489898212254, -0.025839492678642273, -0.03132305666804314, -0.003770459908992052, -0.004160350188612938, 0.030610721558332443]
>,
    "kernel" => #Nx.Tensor<
      f32[128][31]
      [
        [0.19127893447875977, -0.20354685187339783, 0.00715222442522645, -0.20261859893798828, 0.01829296536743641, -0.10723665356636047, 0.0029653224628418684, -0.34728315472602844, 0.21996167302131653, -0.3021117150783539, -0.046828266233205795, -0.0940842479467392, 0.11740834265947342, 0.11471320688724518, 0.013517118990421295, -0.1313537359237671, -0.21643683314323425, 0.1293029487133026, 0.07154353708028793, -0.003663047216832638, -0.31187593936920166, 0.0661044791340828, -0.1209539845585823, -0.2228517085313797, -0.16694411635398865, -0.17859217524528503, -0.20277372002601624, -0.19078464806079865, 0.10752592235803604, -0.3035295307636261, 0.11026991903781891],
        [-0.16017524898052216, -0.08753876388072968, -0.09944022446870804, -0.2616526186466217, -0.12216248363256454, -0.2659483850002289, -0.11005616933107376, -0.02439945936203003, 0.07492928206920624, -0.027908872812986374, 0.12820914387702942, -0.006576727144420147, 0.13019774854183197, 0.06803949922323227, -0.12491944432258606, ...],
        ...
      ]
>
  }
}
```

Done! Now let's see if the classifier works...

## Evaluation

Let's create an input control where we can get an utterance from the user, and then encode 
it into a tensor:

<!-- livebook:{"livebook_object":"cell_input","name":"test sentence","type":"text","value":""} -->

```elixir
utterance = "Я думаю, что моя веб-камера не работает"

test_tensor =
  Req.get!(server <> "?q=" <> URI.encode(utterance)).body
  |> Nx.from_binary({:f, 32})
  |> Nx.reshape({1, dim})
```

```output
#Nx.Tensor<
  f32[1][768]
  [
    [0.035656657069921494, -0.11470914632081985, -0.008336213417351246, 0.04725819081068039, 0.03186924755573273, 0.07852264493703842, 0.08017267286777496, -0.03209451213479042, 0.02732980065047741, -0.04088669270277023, 0.11982234567403793, 0.012704374268651009, 0.004452805500477552, 0.10210815072059631, 0.10588141530752182, 0.04009247571229935, -0.00528173241764307, 0.002549220807850361, 0.09763575345277786, 0.10260072350502014, -0.0015450922073796391, -0.041030801832675934, -0.014938929118216038, -0.024537356570363045, -0.0938982143998146, -0.004252233076840639, 0.11870963871479034, 0.08725015819072723, 0.08703180402517319, 0.07864680886268616, 0.006577798165380955, -0.0030863273423165083, -0.021236510947346687, 0.07108917832374573, 0.08339929580688477, 0.028979865834116936, 9.27539134863764e-4, -0.039742983877658844, 0.008933814242482185, 0.16822251677513123, 0.0839962288737297, 0.028433680534362793, 0.032044652849435806, 0.07510144263505936, -0.010183206759393215, -0.03363952040672302, -0.03731989860534668, -0.12962648272514343, -0.20152674615383148, 0.11135950684547424, ...]
  ]
>
```

And here we call the *predict* function to retrieve the one-hot encoding prediction 
for the feature vector:

<!-- livebook:{"reevaluate_automatically":true} -->

```elixir
require Axon

[index] =
  Axon.predict(model, model_state, test_tensor)
  |> Nx.argmax()
  |> Nx.to_flat_list()

reverse_lookup[index]
```

```output
"faq_webcam_does_not_work"
```
