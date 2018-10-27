---
layout:     post
title:      ASOC之Machine
subtitle:   Linux驱动
date:       2018-10-27
author:     Muggle
header-img:
catalog: 	 true
tags:
    - Linux驱动
---

## 前言 ##
Machine是ASOC架构中的关键部件，没有Machine部件，Codec和Platform是无法工作的。分析内核版本为4.17

## Machine代码分析 ##
以smdk_wm8580.c为例。整体结构是先注册平台驱动,当平台驱动和平台设备的名字相匹配的时候，就会调用平台驱动里的probe函数。<br>
### 1.入口函数：注册平台设备 ###
![](https://i.imgur.com/ztAGdMx.jpg)
<br>
![](https://i.imgur.com/oUa8aOz.jpg)
分配一个名为“soc-audio”的`platform_device`并且赋值给`smdk_snd_device`，将`smdk`结构体里的相关设备信息设置为`smdk_snd_device`这个平台的驱动数据，然后将`smdk_snd_device`加入平台设备列表。`module_init`修饰一下入口函数。

### 2.平台设备驱动 ###
![](https://i.imgur.com/ATFspMi.jpg)
在`soc-core.c`文件中有对应的平台设备驱动，如果匹配成功就会调用`probe`函数。
![](https://i.imgur.com/QKrIB7f.jpg)
此处会调用snd_soc_register_card，会在ASOC core中注册一个card。 此处的card就是smdk_ops结构。接下来谈论此结构的作用。
![](https://i.imgur.com/udyZuMj.jpg)
![](https://i.imgur.com/k3nbDnK.jpg)
其中dai_link结构就是用作连接platform和codec的，指明到底用那个codec，那个platfrom。那是通过什么指定的？  如果有兴趣可以详细看snd_soc_dai_link的注释，此注释写的非常清楚。<br>
	.cpu_dai_name:      用于指定cpu侧的dai名字，也就是所谓的cpu侧的数字音频接口，一般都是i2S接口。如果省略则会使用cpu_name/cou_of_name。
	.codec_dai_name:  用于codec侧的dai名字，不可以省略。
	.codec_name:         用于指定codec芯片。不可以省略。
	.platform_name:     用于指定cpu侧平台驱动，通常都是DMA驱动，用于传输。
	.ops:                         audio的相关操作函数集合。

再次回到snd_soc_register_card函数中，继续分析Machine的作用。
1.  根据struct snd_soc_dai_link结构体的个数，此处是一个，检测下需要设置的name是否已经设置。
if (link->platform_name && link->platform_of_node) {
	dev_err(card->dev,
	"ASoC: Both platform name/of_node are set for %s\n",link->name);
	return -EINVAL;
}
2.  分配一个struct snd_soc_pcm_runtime结构，然后根据num_links，设置card，复制dai_link等。
card->rtd = devm_kzalloc(card->dev,sizeof(struct snd_soc_pcm_runtime) *
			 (card->num_links + card->num_aux_devs),GFP_KERNEL);
    if (card->rtd == NULL)
	    return -ENOMEM;
    card->num_rtd = 0;
    card->rtd_aux = &card->rtd[card->num_links];
 
for (i = 0; i < card->num_links; i++) {
	card->rtd[i].card = card;
	card->rtd[i].dai_link = &card->dai_link[i];
	card->rtd[i].codec_dais = devm_kzalloc(card->dev,
				sizeof(struct snd_soc_dai *) *
				(card->rtd[i].dai_link->num_codecs),
				GFP_KERNEL);
	if (card->rtd[i].codec_dais == NULL)
		return -ENOMEM;
}
3.  然后所有的重点工作全部在snd_soc_instantiate_card函数中实现。

分析snd_soc_instantiate_card函数的实际操作:
1.   根据num_links的值，进行DAIs的bind工作。第一步先bind cpu侧的dai
	cpu_dai_component.name = dai_link->cpu_name;
	cpu_dai_component.of_node = dai_link->cpu_of_node;
	cpu_dai_component.dai_name = dai_link->cpu_dai_name;
	rtd->cpu_dai = snd_soc_find_dai(&cpu_dai_component);
	if (!rtd->cpu_dai) {
		dev_err(card->dev, "ASoC: CPU DAI %s not registered\n",
			dai_link->cpu_dai_name);
		return -EPROBE_DEFER;
	}
此处dai_link就是在machine中注册的struct snd_soc_dai_link结构体，cpu_dai_name也就是注册的name，最后通过snd_soc_find_dai接口出查找。
static struct snd_soc_dai *snd_soc_find_dai(
	const struct snd_soc_dai_link_component *dlc)
{
	struct snd_soc_component *component;
	struct snd_soc_dai *dai;
 
	/* Find CPU DAI from registered DAIs*/
	list_for_each_entry(component, &component_list, list) {
		if (dlc->of_node && component->dev->of_node != dlc->of_node)
			continue;
		if (dlc->name && strcmp(component->name, dlc->name))
			continue;
		list_for_each_entry(dai, &component->dai_list, list) {
			if (dlc->dai_name && strcmp(dai->name, dlc->dai_name))
				continue;
 
			return dai;
		}
	}
 
	return NULL;
}
此函数会在component_list链表中先找到相同的name，然后在component->dai_list中查找是否有相同的dai_name。此处的component_list是在注册codec和platform中的时候设置的。会在codec和platform的时候会详细介绍。在此处找到注册的cpu_dai之后，存在snd_soc_pcm_runtime中的cpu_dai中。

2.  然后根据codec的数据，寻找codec侧的dai。
/* Find CODEC from registered CODECs */
for (i = 0; i < rtd->num_codecs; i++) {
	codec_dais[i] = snd_soc_find_dai(&codecs[i]);
	if (!codec_dais[i]) {
		dev_err(card->dev, "ASoC: CODEC DAI %s not registered\n",
			codecs[i].dai_name);
		return -EPROBE_DEFER;
	}
}
然后将找到的codec侧的dai也同样赋值给snd_soc_pcm_runtime中的codec_dai中。

3.  在platform_list链表中查找platfrom，根据dai_link中的platform_name域。如果没有platform_name，则设置为"snd-soc-dummy"

/* if there's no platform we match on the empty platform */
platform_name = dai_link->platform_name;
if (!platform_name && !dai_link->platform_of_node)
	platform_name = "snd-soc-dummy";
 
/* find one from the set of registered platforms */
list_for_each_entry(platform, &platform_list, list) {
	if (dai_link->platform_of_node) {
		if (platform->dev->of_node !=
		    dai_link->platform_of_node)
			continue;
	} else {
		if (strcmp(platform->component.name, platform_name))
			continue;
	}
 
	rtd->platform = platform;
}

这样查找完毕之后，snd_soc_pcm_runtime中存储了查找到的codec, dai,  platform。

4.  接着初始化注册的codec cache，cache_init代表是否已经初始化过。
	/* initialize the register cache for each available codec */
	list_for_each_entry(codec, &codec_list, list) {
	if (codec->cache_init)
		continue;
	ret = snd_soc_init_codec_cache(codec);
	if (ret < 0)
		goto base_error;
	}

5.  然后调用ALSA中的创建card的函数:  snd_card_new创建一个card
	/* card bind complete so register a sound card */
	ret = snd_card_new(card->dev, SNDRV_DEFAULT_IDX1, SNDRV_DEFAULT_STR1,
		card->owner, 0, &card->snd_card);
	if (ret < 0) {
	dev_err(card->dev,
		"ASoC: can't create sound card for card %s: %d\n",
		card->name, ret);
	goto base_error;
	}

6.  然后依次调用各个子部件的probe函数
	/* initialise the sound card only once */
	if (card->probe) {
	ret = card->probe(card);
	if (ret < 0)
		goto card_probe_error;
	}
 
	/* probe all components used by DAI links on this card */
	for (order = SND_SOC_COMP_ORDER_FIRST; order <= SND_SOC_COMP_ORDER_LAST;
		order++) {
	for (i = 0; i < card->num_links; i++) {
		ret = soc_probe_link_components(card, i, order);
		if (ret < 0) {
			dev_err(card->dev,
				"ASoC: failed to instantiate card %d\n",
				ret);
			goto probe_dai_err;
		}
	}
}
 
	/* probe all DAI links on this card */
	for (order = SND_SOC_COMP_ORDER_FIRST; order <= SND_SOC_COMP_ORDER_LAST;
		order++) {
	for (i = 0; i < card->num_links; i++) {
		ret = soc_probe_link_dais(card, i, order);
		if (ret < 0) {
			dev_err(card->dev,
				"ASoC: failed to instantiate card %d\n",
				ret);
			goto probe_dai_err;
		}
	}
}

7.  在soc_probe_link_dais函数中依次调用了cpu_dai， codec_dai侧的probe函数
	/* probe the cpu_dai */
	if (!cpu_dai->probed &&
		cpu_dai->driver->probe_order == order) {
	if (cpu_dai->driver->probe) {
		ret = cpu_dai->driver->probe(cpu_dai);
		if (ret < 0) {
			dev_err(cpu_dai->dev,
				"ASoC: failed to probe CPU DAI %s: %d\n",
				cpu_dai->name, ret);
			return ret;
		}
	}
	cpu_dai->probed = 1;
}
 
	/* probe the CODEC DAI */
	for (i = 0; i < rtd->num_codecs; i++) {
	ret = soc_probe_codec_dai(card, rtd->codec_dais[i], order);
	if (ret)
		return ret;

8.   最终调用到soc_new_pcm函数创建pcm设备:
	if (!dai_link->params) {
		/* create the pcm */
		ret = soc_new_pcm(rtd, num);
		if (ret < 0) {
		dev_err(card->dev, "ASoC: can't create pcm %s :%d\n",
	dai_link->stream_name, ret);
	return ret;
	}
最中此函数会调用ALSA的标准创建pcm设备的接口:  snd_pcm_new，然后会设置pcm相应的ops操作函数集合。然后调用到platform->driver->pcm_new的函数。此处不帖函数了。

9.  接着会在dapm和dai widget做相应的操作，后期会设置control参数，最终会调用到ALSA的注册card的函数snd_card_register。
	ret = snd_card_register(card->snd_card);
	if (ret < 0) {
	dev_err(card->dev, "ASoC: failed to register soundcard %d\n",
			ret);
	goto probe_aux_dev_err;
	}

总结：  经过Machine的驱动的注册，Machine会根据注册以"soc_audio"为名字的平台设备，然后在同名的平台的驱动的probe函数中，会根据snd_soc_dai_link结构体中的name，进行匹配查找相应的codec, codec_dai，platform, cpu_dai。找到之后将这些值全部放入结构体snd_soc_pcm_runtime的相应位置，然后注册card，依次调用codec, platform，cpu_dai侧相应的probe函数进行初始化，接着创建pcm设备，注册card到系统中。其实ASOC也就是在ALSA的基础上又再次封装了一次，让写驱动更方便，简便。

这样封装之后，就可以大大简化驱动的编写，关于Machine驱动需要做的:
1.   注册名为"soc-audio"的平台设备。
2.   分配一个struct snd_soc_card结构体，然后设置其中的dai_link。对其中的dai_link再次设置。
3.   将struct snd_soc_card结构放入到平台设备的dev的私有数据中。
4.   注册平台设备。




