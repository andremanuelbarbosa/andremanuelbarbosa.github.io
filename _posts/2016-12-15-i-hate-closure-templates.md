---
layout:      post
title:       I hate Closure Templates
description: 
headline:    "I hate Closure Templates"
categories:  [Closure]
tags:        [Closure, Stack Overflow, Java]
image:       
comments:    true
mathjax:     
featured:    false
published:   true
---


I hate Closure Templates! Maybe is due to the lack of documentation available, maybe is because no one else in the world seems to be using this excepts Google (?) or maybe is because I just do not quite understand some of its concepts. When I have to work on a customization or upgrade that involves changing some SOY files, I get depressed. In fact, I think I would rather look at log files full of errors.

## Stack Overflow #1
```java
[WARNING] [talledLocalContainer] java.lang.StackOverflowError
[WARNING] [talledLocalContainer]      at com.google.template.soy.data.SoyMapData.getSingle(SoyMapData.java:192)
[WARNING] [talledLocalContainer]      at com.google.template.soy.data.internal.AugmentedSoyMapData.getSingle(AugmentedSoyMapData.java:157)
[WARNING] [talledLocalContainer]      at com.google.template.soy.data.internal.AugmentedSoyMapData.getSingle(AugmentedSoyMapData.java:161)
[WARNING] [talledLocalContainer]      at com.google.template.soy.data.internal.AugmentedSoyMapData.getSingle(AugmentedSoyMapData.java:161)
[WARNING] [talledLocalContainer]      at com.google.template.soy.data.internal.AugmentedSoyMapData.getSingle(AugmentedSoyMapData.java:161)
[WARNING] [talledLocalContainer]      at com.google.template.soy.data.internal.AugmentedSoyMapData.getSingle(AugmentedSoyMapData.java:161)
```

This is the error shown when accessing the URL with the filter included. I could not find a proper version of the actual source code throwing the error so I have decided the decompile from the core version:

```java
    public SoyData getSingle(String key) {
        SoyData value = super.getSingle(key);
        return value != null?value:this.baseData.getSingle(key);
    }
```

The method call "this.baseData.getSingle" seemed suspicious so I have performed some debugging and confirmed that the "baseData" was itself an instance of the AugmentedSoyMapData class, so the method was calling itself on a recursive loop.

```java
    public SoyData getSingle(String key) {
        SoyData value = super.getSingle(key);
        SoyMapData soyMapData = baseData;
        while (value == null && soyMapData instanceof AugmentedSoyMapData) {
            value = baseData.getMap().get(key);
            soyMapData = ((AugmentedSoyMapData) soyMapData).baseData;
        }
        return value != null ? value : soyMapData.getSingle(key);
    }
```

I have made several changes to this method in order to remove the recursive calls and also to change the parent class SoyMapData. The outcome was something like this:

![Catching](/images/posts/2016-12-15-i-hate-closure-templates/catching.gif "Catching")

## Stack Overflow #2
```java
[WARNING] [talledLocalContainer] java.lang.StackOverflowError
[WARNING] [talledLocalContainer]      at com.google.common.collect.RegularImmutableMap.get(RegularImmutableMap.java:140)
[WARNING] [talledLocalContainer]      at com.google.template.soy.shared.internal.NonpluginFunction.forFunctionName(NonpluginFunction.java:57)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitFunctionNode(EvalVisitor.java:472)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitFunctionNode(EvalVisitor.java:82)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visit(AbstractReturningExprNodeVisitor.java:123)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitGreaterThanOpNode(EvalVisitor.java:375)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitGreaterThanOpNode(EvalVisitor.java:82)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visit(AbstractReturningExprNodeVisitor.java:111)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitExprRootNode(EvalVisitor.java:139)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitExprRootNode(EvalVisitor.java:82)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visit(AbstractReturningExprNodeVisitor.java:81)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visit(AbstractReturningExprNodeVisitor.java:73)
[WARNING] [talledLocalContainer]      at com.google.template.soy.basetree.AbstractReturningNodeVisitor.exec(AbstractReturningNodeVisitor.java:43)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.RenderVisitor.eval(RenderVisitor.java:684)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.RenderVisitor.visitIfNode(RenderVisitor.java:327)
[WARNING] [talledLocalContainer]      at com.google.template.soy.soytree.AbstractSoyNodeVisitor.visit(AbstractSoyNodeVisitor.java:93)
[WARNING] [talledLocalContainer]      at com.google.template.soy.soytree.AbstractSoyNodeVisitor.visit(AbstractSoyNodeVisitor.java:55)
[WARNING] [talledLocalContainer]      at com.google.template.soy.basetree.AbstractNodeVisitor.visitChildren(AbstractNodeVisitor.java:59)
[WARNING] [talledLocalContainer]      at com.google.template.soy.soytree.AbstractSoyNodeVisitor.visitChildren(AbstractSoyNodeVisitor.java:126)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.RenderVisitor.visitBlockHelper(RenderVisitor.java:648)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.RenderVisitor.visitSoyNode(RenderVisitor.java:595)
```

I then looked deep into the closure code, tried several changes in order to fix this second error but all attempts were not successful. Seems like each closure method/declaration is converted into multiple Java method calls, which then leads to a large number of recursive calls. The obvious next step was to look into the actual SOY code and see what was happening there.

```closuretemplate
{template .filters}
    {if $childFilters}
        {if length($childFilters) > $childIndex}

            /* If this is a nested filter or is a placeholder then skip it. */
            {if $childFilters[$childIndex].nested or length(match($childFilters[$childIndex].id, '~choose|~showall')) > 0}
                {call .filters data="all"}
                    {param childIndex: $childIndex + 1 /}
                    {param appliedIndex: length($childFilters) > $childIndex + 1 ?
                        indexOf($appliedFilterIDs, $childFilters[$childIndex + 1].id) : -1 /}
                {/call}

            /* If this is the object type filter parent then pass its children
             * as objectTypeFilters. */
            {elseif length(match($childFilters[$childIndex].id, '~objecttype')) > 0}
                {call .filters data="all"}
                    {param objectTypeFilters: $childFilters[$childIndex].children /}
                    {param childIndex: $childIndex + 1 /}
                    {param appliedIndex: length($childFilters) > $childIndex + 1 ?
                        indexOf($appliedFilterIDs, $childFilters[$childIndex + 1].id) : -1 /}
                {/call}

            /* If this filter is applied then replace the appropriate index of
             * appliedFilters with this filter. */
            {elseif $appliedIndex > -1}
                {call .filters data="all"}
                    {param appliedFilters: concat(concat(slice($appliedFilters, 0, $appliedIndex),
                        $childFilters[$childIndex]),
                        slice($appliedFilters, $appliedIndex + 1)) /}
                    {param childIndex: $childIndex + 1 /}
                    {param appliedIndex: length($childFilters) > $childIndex + 1 ?
                        indexOf($appliedFilterIDs, $childFilters[$childIndex + 1].id) : -1 /}
                {/call}
                
            /* If this filter is not applied then simply append it to
             * unappliedFilters. */
            {else}
                {call .filters data="all"}
                    {param unappliedFilters: concat($unappliedFilters, $childFilters[$childIndex]) /}
                    {param childIndex: $childIndex + 1 /}
                    {param appliedIndex: length($childFilters) > $childIndex + 1 ?
                        indexOf($appliedFilterIDs, $childFilters[$childIndex + 1].id) : -1 /}
                {/call}
            {/if}

        {else}
            {call .compactAppliedFilters data="all"}
                {param appliedIndex: 0 /}
            {/call}
        {/if}

    {elseif $filters and length($filters) > 0}
        {foreach $filter in $filters}
            {if indexOf($appliedFilterIDs, $filter.id) > -1 and not $filter.nested}
                {if $filter.children and length($filter.children) > 0}
                    {call .filters data="all"}
                        {param appliedFilters: $appliedFilterIDs /}
                        {param unappliedFilters: List() /}
                        {param parentFilter: $filter /}
                        {param childFilters: $filter.children /}
                        {param childIndex: 0 /}
                        {param appliedIndex: indexOf($appliedFilterIDs, $filter.children[0].id) /}
                    {/call}

                {else}
                    {call .filtersDispatch data="all"}
                        {param filters: List() /}
                        {param appliedFilters: List() /}
                        {param unappliedFilters: List() /}
                    {/call}
                {/if}
            {/if}
        {/foreach}

    {else}
        {call .filtersDispatch data="all"}
            {param filters: List() /}
            {param appliedFilters: List() /}
            {param unappliedFilters: List() /}
        {/call}
    {/if}
{/template}
```

As expected, there is a large number of recursive calls happening on the .filters template, which calls itself for each child filter in order to loop through the full list of child filters. So I attempted to convert this recursive template into something more flat but this turned into (almost) mission impossible. Because variables are immutable in closure and cannot be re-assigned, I had to create a custom closure function in order to add elements to an existing list.

```java
import com.google.common.collect.Sets;
import com.google.template.soy.data.SoyData;
import com.google.template.soy.data.SoyListData;
import com.google.template.soy.data.restricted.NullData;
import com.google.template.soy.data.restricted.UndefinedData;
import com.google.template.soy.tofu.restricted.SoyTofuFunction;

import java.util.*;
import java.util.List;

public class SelfMergeFunction implements SoyTofuFunction {

    @Override
    public SoyData computeForTofu(List<SoyData> args) {
        final SoyListData self = (SoyListData) args.get(0);
        for (int i = 1; i < args.size(); i++) {
            final SoyData soyData = args.get(i);
            if (!(soyData instanceof UndefinedData) && !(soyData instanceof NullData)) {
                if (soyData instanceof SoyListData) {
                    for (SoyData soyDataValue : (SoyListData) soyData) {
                        self.add(soyDataValue);
                    }
                } else {
                    self.add(soyData);
                }
            }
        }
        return self;
    }

    @Override
    public String getName() {
        return "selfMerge";
    }

    @Override
    public Set<Integer> getValidArgsSizes() {
        return Sets.newHashSet(2, 3, 4, 5, 6, 7, 8, 9);
    }
}
```

I called this function "selfMerge" and then modified the .filters template to be less recursive.

```closuretemplate
{template .filters}
    {if $filters and length($filters) > 0}
        {foreach $filter in $filters}
            {if indexOf($appliedFilterIDs, $filter.id) > -1 and not $filter.nested}
                {if $filter.children and length($filter.children) > 0}
                    {let $objectTypeFiltersTemp: $objectTypeFilters and length($objectTypeFilters) > 0 ? $objectTypeFilters : List() /}
                    {let $objectTypeFiltersUnused}
                        {foreach $childFilter in $filter.children}
                            {if length(match($childFilter.id, '~objecttype')) > 0}
                                {selfMerge($objectTypeFiltersTemp, $childFilter.children)}
                            {/if}
                        {/foreach}
                    {/let}

                    {let $appliedFiltersTemp: $appliedFilters and length($appliedFilters) > 0 ? $appliedFilters : List() /}
                    {let $appliedFiltersUnused}
                        {foreach $childFilter in $filter.children}
                            {let $appliedIndexTemp: indexOf($appliedFilterIDs, $childFilter.id) /}
                            {if $appliedIndexTemp > -1}
                                {selfMerge($appliedFiltersTemp, $childFilter)}
                            {/if}
                        {/foreach}
                    {/let}

                    {let $unappliedFiltersTemp: $unappliedFilters and length($unappliedFilters) > 0 ? $unappliedFilters : List() /}
                    {let $unappliedFiltersUnused}
                        {foreach $childFilter in $filter.children}
                            {let $appliedIndexTemp: indexOf($appliedFilterIDs, $childFilter.id) /}
                            {if not($childFilter.nested or length(match($childFilter.id, '~choose|~showall')) > 0) and not(length(match($childFilter.id, '~objecttype')) > 0) and not($appliedIndexTemp > -1)}
                                {selfMerge($unappliedFiltersTemp, $childFilter)}
                            {/if}
                        {/foreach}
                    {/let}

                    {call .compactAppliedFilters data="all"}
                        {param parentFilter: $filter /}
                        {param appliedFilters: $appliedFiltersTemp /}
                        {param objectTypeFilters: $objectTypeFiltersTemp /}
                        {param unappliedFilters: $unappliedFiltersTemp /}
                        {param appliedIndex: 0 /}
                        {param hideFilterGroup: $hideFilterGroup /}
                        {param itemsView: $itemsView /}
                        {param urlParams: $urlParams /}
                        {param urlPath: $urlPath /}
                    {/call}
                {else}
                    {call .filtersDispatch data="all"}
                        {param filters: List() /}
                        {param appliedFilters: List() /}
                        {param unappliedFilters: List() /}
                        {param hideFilterGroup: $hideFilterGroup /}
                        {param itemsView: $itemsView /}
                        {param urlParams: $urlParams /}
                        {param urlPath: $urlPath /}
                    {/call}
                {/if}
            {/if}
        {/foreach}
    {else}
        {call .filtersDispatch data="all"}
            {param filters: List() /}
            {param appliedFilters: List() /}
            {param unappliedFilters: List() /}
            {param hideFilterGroup: $hideFilterGroup /}
            {param itemsView: $itemsView /}
            {param urlParams: $urlParams /}
            {param urlPath: $urlPath /}
        {/call}
    {/if}
{/template}
```

This time, the outcome was more like this:

![Smash Computer](/images/posts/2016-12-15-i-hate-closure-templates/smash-computer.gif "Smash Computer")

## Stack Overflow #3

```java
[WARNING] [talledLocalContainer] java.lang.StackOverflowError
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.resolveDataRefFirstKey(EvalVisitor.java:685)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitDataRefNode(EvalVisitor.java:207)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitDataRefNode(EvalVisitor.java:82)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visit(AbstractReturningExprNodeVisitor.java:94)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visit(AbstractReturningExprNodeVisitor.java:73)
[WARNING] [talledLocalContainer]      at com.google.template.soy.basetree.AbstractReturningNodeVisitor.visitChildren(AbstractReturningNodeVisitor.java:63)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visitChildren(AbstractReturningExprNodeVisitor.java:137)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitFunctionNode(EvalVisitor.java:489)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitFunctionNode(EvalVisitor.java:82)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visit(AbstractReturningExprNodeVisitor.java:123)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitEqualOpNode(EvalVisitor.java:411)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitEqualOpNode(EvalVisitor.java:82)
[WARNING] [talledLocalContainer]      at com.google.template.soy.exprtree.AbstractReturningExprNodeVisitor.visit(AbstractReturningExprNodeVisitor.java:116)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitAndOpNode(EvalVisitor.java:428)
[WARNING] [talledLocalContainer]      at com.google.template.soy.sharedpasses.render.EvalVisitor.visitAndOpNode(EvalVisitor.java:82)
```

Back to the drawing board. A deeper look into the same SOY file showed another potential problem on .filter template, with a few recursive calls too.

```closuretemplate
{template .filter}
    {if $type == 'composite' and $children and length($children) > 0}
        {if length($children) == 1 and $children[0].type != 'simple'}
            {call .filter data="$children[0]"}
                {param appliedFilterIDs: $appliedFilterIDs /}
            {/call}

        {elseif typeof($childIndex) != 'number'}
            {call .filter data="all"}
                {param childIndex: 0 /}
            {/call}

        {elseif indexOf($appliedFilterIDs, $children[$childIndex].id) > -1}
            {call .compositeFilter data="all"}
                {param appliedFilter: $children[$childIndex] /}
            {/call}

        {elseif $childIndex < length($children) - 1}
            {call .filter data="all"}
                {param childIndex: $childIndex + 1 /}
            {/call}

        {else}
            /* Base case: render a composite filter with no child filters
             * displayed. */
            {call .compositeFilter data="all"}
                {param appliedFilter: $children[0] /}
            {/call}
        {/if}

    {elseif $type != 'simple' and $type != 'composite'}
        {call .dynamicFilter data="all" /}
    {/if}
{/template}
```

Using the same approach as before, I was able to reduce the number of recursive calls on that template:

```closuretemplate
{template .filter}
    {if $type == 'composite' and $children and length($children) > 0}
        {let $unprocessedFilters: List() /}
        {foreach $child in $children}
            {if indexOf($appliedFilterIDs, $child.id) > -1}
                {call .compositeFilter data="all"}
                    {param appliedFilter: $child /}
                {/call}
            {elseif $child.type != 'simple'}
            {else}
                {let $unprocessedFiltersUnused}
                    {selfMerge($unprocessedFilters, $child)}
                {/let}
            {/if}
        {/foreach}

        {if length($children) == length($unprocessedFilters)}
            {call .compositeFilter data="all"}
                {param appliedFilter: $children[0] /}
            {/call}
        {/if}
    {elseif $type != 'simple' and $type != 'composite'}
        {call .dynamicFilter data="all"}
            {param appliedFilterIDs: $appliedFilterIDs /}
        {/call}
    {/if}
{/template}
```

This is just another example and I had to convert each template which is executed during the rendering of these filters into something less recursive.

![Cellebrations](/images/posts/2016-12-15-i-hate-closure-templates/cellebrations.gif "Cellebrations")

## Stack Overflow #999
Just kidding but there were several other issues too. First, the data="all" no longer seems to work after the changes on the .filters template, so I had to change all templates down the chain to pass each parameter. This means that I had to change for example the pagination template, which has nothing to do with the actual filters issue.

This fix seems to work with up to 2 filters in the URL but if we had a 3rd then the Stack Overflow error shows up again. I think this has nothing do with the number of filters but with the overall size of the combined filters. I have tried to change some templates further but there was nothing else that could be changed in terms of less recursive calls without creating the entire rendering from scratch. While the fix seems to be working now, there is not guarantee that will continue to work as the number of values in the Profile Field increase.