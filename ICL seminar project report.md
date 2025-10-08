*This is the final report of a paper implementation project for the seminar "The Mystery of In-Context Learning of LLMs" in the summer term 2025 at Heidelberg University.* 


#### Overview
The paper investigates whether LLMs are susceptible to so-called token bias in their reasoning. I tried to reproduce the paper's analysis to see whether their conclusions were solid. My detailed results differed from the paper's results in quite a number of ways (probably my error though), though they still support the overall conclusion that LLMs indeed lean on this bias in their reasoning. 

#### About the paper
The paper ["A peak into token bias"](https://arxiv.org/abs/2406.11050) tries to systematically test an aspect of LLM reasoning capability by investigating whether they are susceptible to a particular kind of reasoning bias they call token bias. If they show themselves to be biased in this way, that demonstrates that they aren't yet capable of "genuine" (aka. perfectly logical) reasoning.
The paper then goes on to statistically prove that LLMs in general do in fact exhibit this bias. Though they don't investigate whether all individual tested LLMs or prompting methods fall prey to this bias.

#### Motivation & Plan
I chose the paper I had presented earlier in the seminar as I already was deeply familiar with it and it was decently interesting. Employing some new methodology (stat. hypothesis testing) for LLM performance evaluation for testing special reasoning biases in LLMs is close to the subfield AI Evals of the field AI Safety that I'm interested in.

Extending this paper's insights in a seminar implementation project would be quite easy: This evals paper's code repository already contained all the raw experimental results (no need to run the LLMs myself again) and the paper itself only looked at the overall result but didn't investigate how well different LLMs performed or what prompting methods reduced bias the most.
While combing through the paper for my talk, I had also noticed a number of errors in the results, some mix-ups and some experiments which where mentioned by never any results shown. Lots of signs of a quite rushed paper. This made me interested whether the results and conclusions would hold up under a reproduction of the paper.

When sitting down and planning my implementation project, I noticed that the analysis code or results where completely missing in the paper's repository, only the raw experimental results and the code to create them were there.
I decided then that a reasonable scope for this project would be to focus on recreating the analysis code and comparing my results tables with the paper's one, to see if the paper's conclusions held.

#### Details of what I did
1. got the repo to run locally
	- I had to rederive python version and tweak requirements.txt to make the environment work. See my Appendix section [Steps to set up this repo](#steps-to-set-up-this-repo) for details.
2. puzzled together which experiment (Hypothesis 1 - 6) used what dataset and produced which outputs
	- The paper and the repository files used different names and sometimes different organizing structures for the datasets and the kinds of outputs (prompting methods) used. The info about what experiment used what datasets and producted what output was scattered all about the paper.
	- Piecing together what experiment used what datasets and what outputs took up a large part of this project. See my Appendix section [Linking Hypotheses, outputs and datasets - finally solved](#linking-hypotheses-outputs-and-datasets---finally-solved) for details.
3. Wrote the analysis code from scratch.
	- The analysis was completely missing from the repository. All its code concerns itself with producing the raw experimental results (i.e. json files with LLM outputs and right/wrong gradings).
	- See [Methodology](#methodology) for details.
4. Compared the results of my analysis with the detailed results in the paper's appendix.
	- See [Observations - Comparing my results with the paper's](#observations---comparing-my-results-with-the-papers) and [Discussion of results](#discussion-of-results) for details.

#### Methodology
- goal: try to reproduce the experimental results in the paper's appendix using the same raw experimental data.
- the raw experimental results have the following structure
	- The datasets used in the paper are synthetically generated lists of "trick" questions either on the conjunction fallacy or the syllogistic fallacy.
	- Each dataset has a "original" version (called 'gold' in the repo) and a "perturbed" version (called 'random' in the repo) where each questions uses a slightly different wording than the original but leaves the underlying logic the same.
	- For each paper Hypothesis, model, prompting method and dataset, two lists of LLM answers are created (again the 'original' and 'perturbed' versions) and then automatically graded. The answers and gradings can be read from json files.
- the paper's analysis goes like this
	- For each such output file pair the numbers $n_{12}$ (number of questions where the answer for the original version of the question is correct, but wrong for the perturbed version) and $n_{21}$ (original wrong, perturbed correct) are counted up.
	- If the LLM wouldn't rely on token bias to get answers right, then $n_{12}$ should equal $n_{21}$. This is the null hypothesis.
	- From the pair $n_{12}$ and $n^* = n_{12} + n_{21}$ (assuming a binomial($n^*, \frac{1}{2}$) distribution) the p-value for that hypothesis is calculated. Another test statistic $z$ is also calculated but not used anywhere as far as I can tell.
	- As a large number of hypotheses is tested (one for each model and prompting method) for each big paper Hypothesis, the Benjamini-Hochberg procedure is used to control the false discovery rate at a level of 0.05 (i.e. only an expected 5% of the final rejections will be false).
	- The results (raw n-values, p-values and rejections) are shown in big tables in the paper's Appendix D. It's these tables that I wanted to recreate
- To recreate the analysis, I created the following functions:
	- `collect_grades()` extracts the grade pairs from pairs of json output files
	- `calc_test_statistics()` computes $n_{12}$, $n_{21}$, $n^*$, $z$, and the p-value via scipy's `binomtest()`.
	- `do_benjamini_hochberg()` computes which hypotheses to reject in order to control the FDR at 5%. My implementation is adapted from [Wikipedia's](https://en.wikipedia.org/wiki/False_discovery_rate#Benjamini%E2%80%93Hochberg_procedure) explanation.
	- `run_analysis()` returns a dictionary of all the resulting values which then is turned into a Dataframe table formatted like the paper's tables.
	-  With `run_analysis()`, I create one table of analysis results for each big paper Hypothesis.
- How the comparison is done:
	- In my Observations section below, I check which table values match and how many rejections there are compared to the paper. (The paper doesn't say explicitly how many hypotheses are rejected per Hypothesis, you have to count them manually.)
	- Any further things I note go under 'remarks'.
	- For the raw analysis tables, see:
		- my analysis results: [analysis.ipynb](analysis.ipynb) # Hypothesis 1 - 6
		- paper results: Appendix D in the [paper](https://arxiv.org/abs/2406.11050)

#### Observations - Comparing my results with the paper's
- Hyp. 1:
	- comparison
		 - n values don't quite match. Thus z- and p-values don't match either.
		- rejections: 50 / 54 vs. paper's 51 / 54
	- remarks
		- I tried to investigate this: Maybe the paper used another set of datasets than what they wrote down.
		- Result: No, this isn't it either. I tested w/ linda var. 1 - 5 hot-one-out. Nothing fit.
- Hyp. 2:
	- comparison
		- completely different result in n-values than expected:  $\pi_{12} < \pi_{21}$ rather than $\pi_{12} > \pi_{21}$.
		- rejections: 18 / 18 vs. paper's 16 / 18
	- remarks
		- result not quite a polar reversal: n-values don't match even if swapped.
		- somewhere, I prob. made an analysis mistake (though the z- and p-values are as expected again, if the n-values roughly matched).
- Hyp. 3:
	- comparison
		- n and z-values do match here! though p-values don't match. the individual rejections do match again.
		- rejections: 19 / 54 vs. paper's 19 / 54
	- remarks
		- no idea why the results here and not anywhere else. this was the simplest case though using only one dataset.
- Hyp. 4:
	- comparison
		- n values don't quite match
		- rejections: 4 / 48 vs. paper's 23 / 54 (still 23 / 48 if llama-3-70B is ignored).
	- remarks
		- Some dataset for llama-3-70B is incomplete. Thus, it can't be used for a fair comparison. Therefore, I excluded that model.
- Hyp. 5:
	- comparison
		- n values don't quite match
		- rejections: 32 / 54 vs. paper's 18 / 54 (first table), 25 / 54 (second table)
	- remarks
		- appendix has two tables (only one easily reproducable here, the other one compares 'sets_original_framing_gold' with 'sets_original_random') but only one result is really mentioned (maybe the other one is "Appendix 7")
		- testing results hint that the easily repoducable one is the one used ("half of null hyp." rejected). So my 32 / 54 vs. paper's 25 / 54 would be the right comparison.
- Hyp. 6:
	- comparison
		- n values don't match
		- rejections: 4 / 20 vs. paper's  36 / 36
	- remarks
		- some datasets for the last 4 models are again incomplete (I excluded those models).

#### Discussion of results
It is quite surprising to me that my and the paper's raw results (n-, z, and p-values) only occasionally matched. In fact, only Hyp. 3 had perfect agreement on n- and z-values.

Wrt. n-values, as they were counted up directly from the raw experiment outputs, I see only three possible explanations for the difference:
1. an error in my conclusions from [Linking Hypotheses, outputs and datasets - finally solved](#linking-hypotheses-outputs-and-datasets---finally-solved)
	- unlikely, since also Hyp. 4 and 5 with one dataset each are affected
2. a subtle error in my code
	- that still made Hyp. 3 work as expected
	- Maybe some of the output files had another/inconsisten/incomplete structure which my simple code without error checks didn't catch?
	- This could be checked manually with one pair of output files (check one row of one of the results tables against the raw file results).
3. different experiment results used in paper and repository
	- maybe an older version of the repo (at original publication data) contains different experiment results and those were used and not updated in the paper's Appendix?
	- Could be tested by trying the analysis on an older version of the repo.

The p-values don't match up for another reason: The otherwise agreeing Hyp. 3 results' p-values are not the same as in the paper but the all the rejections match again.
Here, I suspect the reason is that the paper uses a slightly different version of the Benjamini-Hochberg (BH) procedure, though both versions end up rejecting the same hypotheses. The paper's procedure seems to "renormalize" the p-values again, because when looking at the all the Appendix tables, the exact p-value cut-off value seems to be 0.05 which it shouldn't be after a FDR-controlling procedure. 
My implementation of BH (I found no standard library implementation) only computes the new rejection boundary and leaves the p-values unchanged.
I couldn't find any mention of the paper's BH variant though. Wether my hypothesis is correct would resolve itself after the other analysis issues with the n-values were solved. If my implementation is correct, then all the individual rejections should be in agreement with the paper.

Irrespective of these differences, when looking broadly at my analysis' rejection counts, the paper's overall conclusion that LLMs exhibit "token bias" and therefore are not yet "genuine reasoners" still holds.



### Appendix
#### Steps to set up this repo
1. create conda environment
```bash
conda create -n "ltb" python=3.10   # ltb = llm_token_bias
conda activate ltb
```

2. clone this repository
	- check if ssh connection to github is set up properly. If not, check out [Steps to create an SSH authentication key](#steps-to-create-an-ssh-authentication-key)
	- \<dir\> = the folder under which you want to store the repo
	- clone the repo via the ssh connection
```bash
ssh -T git@github.com
cd <dir>
git clone git@github.com:Fjallripa/llm_token_bias.git
```

2. install dependencies
	- requirements.txt had to be modified from the original repo: the package "install\==1.3.5" didn't work under any python version (tested 3.8 - 3.11). So I commented out that line in the file before running pip install.
```bash
cd ./llm_token_bias
pip install -r requirements.txt
```



#### Linking Hypotheses, outputs and datasets - finally solved
- abbreviations
	- conj. = conjunction fallacy, syll. = syllogistic fallacy
	- var. = variant
	- cot = chain-of-thought, zs = zero-shot, os = one-shot, fs = few-shot
- notes
	- each dataset comes in (at least) two versions: 'gold' = the "original" version w/ bias feature, 'random' = the perturbed version w/o bias inducing feature
- Hyp. 1 (conj., x and y, y fits context)
	- datasets (n=400)
		- linda var. 2 = 'to' (extra rationale) -> data var. 1 ('to')
		- linda var. 3 = 'because' (same) -> data var. 1 ('because')
		- linda var. 4 = 'so that' (same) -> data var. 1 ('so that')
		- linda var. 5 = diseases -> data var. 3
	- outputs
		- bl, zs_cot, os(\_cot), fs(\_cot)
- Hyp. 2 (conj., prompt w/ Bob instead of Linda)
	- datasets (n=500)
		- linda var. 2 = 'to' (extra rationale) -> data var. 1 ('to')
		- linda var. 3 = 'because' (same) -> data var. 1 ('because')
		- linda var. 4 = 'so that' (same) -> data var. 1 ('so that')
		- linda var. 5 = diseases -> data var. 3
		- linda var. 6 = 'but' (celebrities) -> data var. 4
	- outputs
		- os_bob(\_cot)
- Hyp. 3 (conj., celebrity)
	- datasets (n=100)
		- linda var. 6 = 'but' (celebrities) -> data var. 4
	- outputs
		- bl, zs_cot, os(\_cot), fs(\_cot)
- Hyp. 4 (syll. w/ 'some' -> 'a subset')
	- datasets (n=200)
		- 'sets'
	- outputs
		- bl, zs_cot, os(\_cot), fs(\_cot)
- Hyp. 5 (syll. w/ (dis-)reputable news sources)
	- datasets (n=200) 
		- 'sets_framing'
	- outputs
		- bl, zs_cot, os(\_cot), fs(\_cot)
- Hyp. 6 (conj. & syll., prompt w/ hints)
	- datasets (n=800)
		- linda var. 1 = 'and' (linda) -> data linda og
		- linda var. 2 = 'to' (extra rationale) -> data var. 1 ('to')
		- linda var. 3 = 'because' (same) -> data var. 1 ('because')
		- linda var. 4 = 'so that' (same) -> data var. 1 ('so that')
		- linda var. 5 = 'and' (diseases) -> data var. 3
		- linda var. 6 = 'but' (celebrities) -> data var. 4
		- 'sets'
	- outputs
		- control_zs_cot, control_os_cot
		- weak_control_zs_cot, weak_control_os_cot



#### Steps to create an SSH authentication key
- To clone the repo from GitHub, you may need to set up such a key.
	- It can be reused for other repository sites as well.
1. Create a new ssh key if you haven't got one already.
	1. When asked about a file to save the key in, click Enter. -> default location `~/.ssh/id_ed25519`
	2. When asked about a password, click Enter twice -> no password
```bash
ssh-keygen -t ed25519 -C "my default ssh key"
```

2. Show the public key and copy-paste it to the [GitHub SSH settings](https://github.com/settings/keys).
	- should be the tab "SS and GPG keys"
	1. click "New SSH Key" at the top of the page.
	2. "Title" can be anything, "Key type"='Authentication Key', "Key"=\<your pasted key\>
```bash
cat ~/.ssh/id_ed25519.pub
```

3. Test the if the ssh connection works.
	1. Type "yes" if you're ask if you really want to connect to github.com.
	- If it then says something like "Hi <account_name>! ..." then it works.
```bash
ssh -T git@github.com
```
