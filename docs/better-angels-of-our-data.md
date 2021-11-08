

At EMNLP 2021, [Crowdsourcing Beyond Annotation:Case Studies in Benchmark Data Collection](https://nlp-crowdsourcing.github.io/) is one of the tutorials available to conference goers. Although I am not attending the conference, I took a look at the slides via a [Tweet from Yoav Artzi/Sam Bowman](https://twitter.com/yoavartzi/status/1456683247384080393). Whilst the slides do a a nice job at describing a variety of case studies using mechanical turk for data collection, the lack of alternatives offered up made it feel like crowdsourcing is the only option for data collection in NLP. Perhaps this was not the impression given off in person, and certainly I think the tutorial is useful - just that it lacked alternatives.



## Thoughts on Crowdsourcing

### Crowdsourcing good data is very difficult

As we have seen over the last couple of years, it is very easy to crowdsource data which has substantial problems. In case you are not familiar, here are some of the papers discussing the subject of annotation artifacts and annotator biases in crowdsourced datasets:

- [Annotation Artifacts in Natural Language Inference Data](https://aclanthology.org/N18-2017/)
- [Investigating Biases in Textual Entailment Datasets](https://arxiv.org/abs/1906.09635)
- [Hypothesis Only Baselines in Natural Language Inference](https://aclanthology.org/S18-2023/)
- [Are We Modeling the Task or the Annotator? An Investigation of Annotator Bias in Natural Language Understanding Datasets](https://arxiv.org/abs/1908.07898)

If you consider that some people have spent their entire research agenda dedicated to how to perform effective crowdsourcing for NLP tasks - it begins to look like an area in which real expertise is required and is not something that should be approached lightly. One such person is [Julian Michael](https://julianmichael.org/). If you are interested in rock solid crowdsourcing for novel NLP tasks (in particular, QA SRL), I strongly recommend checking out some of Julian's papers, in particular:

- [Controlled Crowdsourcing for High-Quality QA-SRL Annotation](https://aclanthology.org/2020.acl-main.626/)
- [Crowdsourcing Question-Answer Meaning Representations](https://aclanthology.org/N18-2089/)
- [AmbigQA: Answering Ambiguous Open-domain Questions](https://aclanthology.org/2020.emnlp-main.466/)


### Crowdsourcing is not fun work

Managing a crowdsourcing task is not a fun job. It's difficult, people will annotate things incorrectly, people will write scripts to automate their responses, people won't understand the task, people will be pisssed off when they don't get paid. Your automated bonus calculation code won't work and you'll have to do it manually. You'll have to complain to the platform when your data is mis-parsed. You'll inevitably have to kick people off your task for taking shortcuts. Do you really want to have to tell someone you've been paying 10 cents per item of work that the way they've been doing it more efficiently so as to earn 1/5th of a living wage that you won't be paying them because they've "cheated"?


### Crowdsourcing MNLI for $60,000

One aspect of the tutorial slides that really stood out to me was that the overall cost for the MNLI dataset was \$60,000. I would have guessed lower than this as a ballpark figure, but this order of magnitude is relatively standard (e.g whilst I was at AllenNLP, the DROP dataset cost ~$20k to crowdsource).

Taking a step back, does this actually represent value for money? I think certainly not in the case of SNLI. For this much money, certainly in Europe, you could employ two university graduates for a year to annotate a dataset. Assuming they could work at a rate of ~500 examples per day, you could end up with a dataset of 250k (presumably much higher quality) examples just by waiting for a year. This also frees up **your** time, which is important.



## Some alternatives

So, if crowdsourcing is out the window, what other options are there? Well, it turns out there's quite a few. Some of them are even fun and interesting!

### Annotate data yourself :scream:

Data annotation is not actually so impossibly slow/mundane that you can't do it yourself (and if you do think that, maybe check yourself). I have annotated a dataset of sentences from a legal corpora before by hand, which took me about 2 weeks of occasional work, maybe an hour or so per day. I ended up with a dataset of 2.5k fully annotated sentences, which I used to train an NER model which obtained accuracies in the high 80s/90s in terms of F1.

Also, data annotation is extremely helpful for exposing yourself to actual data, and the huge number of edge cases, context considerations and weirdness that comes with actual human language. It also makes you think about niche design decisions regarding the annotation process itself. You, as a modeler of this data as well as the annotator, *know a lot about how the data will be used and how your model will process it*, which is extremely important. You can read about an example of this in another of my [blog posts](./numeric_annotation.md).

If you are creating a dataset on Mechanical Turk, I would strongly recommend that you spend *at least* 2 days just annotating data using the same interface as the crowdworkers you are employing. It will be helpful for the above reasons, but also for other practical things like working out fair pay rates and how much throughput you can expect if the task is being done correctly.

### Work with annotators to build smart interfaces

NLP researchers are often so focused on *what* data they want to collect that there isn't much time for stopping and thinking about *how* you can collect it more efficiently. There are a miryad of ways to speed up data annotation, many of which are technically very interesting! Active learning, semi-supervised learning, bulk annotation, adversarial data collection (which the tutorial touches on) and other methods can all be applied with humans in the loop. Just as importantly as the technical/modeling side is the Annotator Experience.

If you are a PhD student in ML/NLP, it can be a nice break/a refreshing change of stimulus to learn how to build good frontends for your models. It's an extremely valuable skill (your advisor will always love a demo of your work to showoff at talks etc) which is very transferable across industries and roles. It's also a very different style of programming from what you may be used to, and the feedback loop is blissfully short, particularly compared to waiting N hours/days/months for your ML models to train.

Just before I left AI2, I worked with 2 in house data annotators for a project for labeling PDFs. There are actually a lot of interesting problems in designing interfaces for data heavy tasks such as annotation. They are also not always what you might expect - in my experience, good annotators want 3 main things:

- Keyboard shortcuts for labeling, "undo/redo" functionality
- Good keyword search and "label all"
- Easy navigation of their annotation history, so they can correct/reconsider/reference previous annotations

Implementing CMD-Z in a web app is actually quite challenging, because it requires you to keep track of all modifications to the state of your annotation UI, so that you can revert them back to some previous user state. There is a reason that not many web applications have this functionality :)


### Do research that uses implicit data

Another option for annotation is to simply do research which does not require explicit manual annotation. I find that papers which have used some creative trick to find a source of labeled data, they are usually more interesting, even if the data has some quirks. Here are a couple of ones which stick out:

- [Querent Intent in Multi-Sentence Questions](https://arxiv.org/abs/2010.08980) - A very cool dataset collected by identifying that multi-sentence questions can be categorised into several types of "question discourse relations". These multi-sentence questions are easy to collect in the wild (do two consecutive sentences contain question marks, as an extremely crude approximation). By adding this type of discourse structure, this dataset leads the way for interesting generative modeling research, like controlled follow-up question generation.

- [Where’s My Head? Definition, Dataset and Models for Numeric Fused-Heads Identification and Resolution](https://arxiv.org/abs/1905.10886) - Numeric fused heads are noun phrases (typically quantities) which implicitly refer to a head noun which is not stated in the text (e.g "It's worth about 2 million" (pounds)). Although this dataset does actually include mechanical turk annotation, a large amount of effort was directed into it's pre-construction, i.e mining sentences which contain possible fused heads. They do this simply by looking for sentences which contain a a numeric noun phrase but no other nouns, plus a few rules to catch mistakes from parsers.

More generally, there are many tasks which have naturally occuring data which is ripe for collection and extension, such as summarization. Extending automatically collected resources with additional metadata may not seem like the most glamorous of papers, but it actually provides many nice avenues into identifying weaknesses in existing models through faceted evaluation.


### Conclusion

Overall, I found the  EMNLP tutorial slides informative and the case studies quite interesting - I simply want to raise the idea that crowdsourcing is pretty flawed as data collection method (unless you are really an expert at it specifically), not actually very fun to do, and not particularly good for the people doing it.

Whilst I don't doubt that crowdsourcing has its uses for "automated" evaluation and gathering "ordinary" (in the supreme court sense) judgements from non-researchers, I think that there are better ways to spend your days, where you can learn more, have more fun, and not contribute to the creation of a weird, data underclass in our society. With advances in modeling techniques over the last couple of years combined with creative or unusual methods for curating naturally occurring textual data, doing interesting/different NLP work is now easier than ever. Thank you for attending my TED talk.
