# HowTo - Tester un composant React

## Table des matières

  1. [Introduction](#introduction)
  1. [Tests de composants](#tests-de-composants)
  1. [Annexe: trace du cycle de vie d'un composant](#annexe-trace-du-cycle-de-vie-d-un-composant)


## Introduction
Pour faciliter le test de composants React, Facebook fournit un utilitaire appelé `ReactTestUtils`.
Il s'utilise en complément des frameworks de tests tel que Jasmine.

Nous allons voir l'ensemble des problèmes que posent l'utilisation de cette librairie



## Tests de composants

Il est important de bien connaitre le cycle de vie d'un composant React pour le tester correctement.

### Cycle de vie d'un composant React

Un composant React a un cycle de vie **en trois phases**: 

* création du composant
* mise à jour du composant
* destruction du composant

A chacune de ces phases, vont s'exécuter un ensemble de méthodes systèmes React. Les méthodes

* création du composant: `getInitialState`, `getDefaultProps`, `componentWillMount`, `componentDidMount`
* mise à jour du composant: `componentWillReceive`, `PropsshouldComponentUpdate`, `componentWillUpdate`, `componentDidUpdate`
* destruction du composant: `componentDidUnmount'

La **création** du composant s'effectue lors du **premier appel** à la méthode render du composant parent, encore appelé container.

La **mise à jour** du composant s'effectue à **partir du deuxième appel** de la methode render du container.

Il y aura **recréation** du composant, si l'élément react se trouve dans un **nouveau container** parent.


### Fonctionnement de la methode renderIntoDocument

ReactTestUtils utilise la fonction `renderIntoDocument` pour effectuer des tests unitaires?

Voyons le fonctionnement interne de cette fonction.

Cette méthode prend en paramètre un `ReactElement`, pour simplifier un objet sous la forme `JSX` suivant:
```javascript
<MyComponent param1={value1} param2={value2} .../>
```

L'implémentation de cette fonction est très simple :

```javascript
  renderIntoDocument: function( aReactElement) {
    var div = document.createElement('div');
    return ReactDOM.render( aReactElements, div);
  }
```
  
Cette fonction effectue les actions suivantes:

* création d'un react élement parent
* appel de la methode render du react élement parent, en lui passant ses react éléments fils
* retourne le react component fils nouvellement crée
  
Comme le react **element parent** est un **nouvelle objet** à chaque appel de la methode `render`, cela signifie que cet élement n'a aucun contenu quand la methode render est appliqué sur lui.
Par voie de conséquence, l'appel à cette méthode `render` va créer de nouveaux composants react fils  correspondant à la spécification donnée par aReactElements.

On en conclue que les appels successifs de la methode `renderIntoDocument`:

* ne mettent **pas en phase de mise à jour** le  composant à testé (deuxième phase) 
* ne permet **pas de tester le changement de props** d'un composant déja crée
* les **methodes** componentWillReceiveProps, shouldComponentUpdate, componentWillUpdate, componentDidUpdate ne seront **jamais appelées** par l'appel de la methode renderIntoDocument. voir l'annexe 1 montrant les traces d'appel de differentes methodes render.
  
Ces contraintes ne sont pas génantes pour effectuer des tests unitaire, mais ne conviennent pas pour effectuer des tests de composants ou il faut tester le composant dans ses differents contexts d'utilisation réel.


### Résolution du problème

La solution est de ne pas recreer à chaque appel de la methode render un nouveau container parent, mais au contraire passer le container parent crée lors du premier appel aux autres appels de la methode render.

On va donc splitter la fonction `renderIntoDocument` en deux étapes : 

* **création du container**
* **appel de la methode render** avec le **container** juste crée pour **parent**

```javascript
// create the container
let container = document.createElement('div');

// call render method
let component = ReactDOM.render( reactElement, container);
```

On pourra écrire nos tests de la facon suivante 

```javascript
  let parent = document.createElement('div');

  let componentToTest = ReactDOM.render(<MyComponent props1={value1} .../>, parent);
  ...
  ReactDOM.render(<MyComponent props1={value2} .../>, parent);  // update props
  ...
  ReactDOM.unmountComponentAtNode(parent);                      // free elements from parent
```

et encore mieux en utilisant le `beforeEach`et `afterEach` de `jasmine`.

Exemple:

```javascript
var node;
var toTest;

describe('Tabs.spec', () => {
  beforeEach(()=> {
    node = document.createElement('div');
  });

  afterEach(()=> {
    ReactDOM.unmountComponentAtNode( node);
  });

  it('Check creation, navigation and contents', ()=> {
    toTest = ReactDOM.render(<Tabs firstTab={1} tabList={config.tabList}/>, node);
    expect(toTest.getCurrentTab()).toEqual(1);
    expect(toTest.getTabContent()).toEqual( config.tabList[1].content);
    ReactDOM.render(<Tabs firstTab={2} tabList={config.tabList}/>, node);
    expect(toTest.getCurrentTab()).toEqual(2);
    expect(toTest.getTabContent()).toEqual( config.tabList[2].content);
  });
  ...
```

Remarques: 

* Comme dans les TU, on pourra continuer à utiliser les autres méthodes de `ReactTestUtils`.
* On pourra aussi utiliser dans les TU, ReactDOM.render à la place de TestUtils.renderIntoDocument


## Annexe: Trace du cycle de vie d'un composant

Le programme consiste a appeler deux fois une methode render sur un react element, avec à chaque fois un changement de props

Il y a deux cas testés :

* Utilisation de la methode `ReactDOM.render`
* Utilisation de la methode `TestUtils.renderIntoDocument`

```javascript
 it('Lifecycle of a JSX react component used via ReactDOM.render method', ()=> {
    let node = document.createElement('div');
    ReactDOM.render(<Comp props1={'value1'}/>, node);
    ReactDOM.render(<Comp props1={'value2'}/>, node);
    ReactDOM.unmountComponentAtNode(node);
  });

  it('Lifecycle of a react component used via ReactDOM.render method', ()=> {
    let node = document.createElement('div');
    ReactDOM.render(React.createElement(Comp, {props1: 'value1'}), node);
    ReactDOM.render(React.createElement(Comp, {props1: 'value2'}), node);
    ReactDOM.unmountComponentAtNode( node);
  });

  it('Lifecycle of a react component used via ReactTestUtils.renderIntoDocument method', ()=> {
    let toTest1 = TestUtils.renderIntoDocument(<Comp props1={'value2'}/>);
    let toTest2 = TestUtils.renderIntoDocument(<Comp props1={'value2'}/>);
    ReactDOM.unmountComponentAtNode(ReactDOM.findDOMNode(toTest1).parentNode);
    ReactDOM.unmountComponentAtNode(ReactDOM.findDOMNode(toTest2).parentNode);
  });

```
Le résultat est le suivant :

```javascript
Lifecycle of a react component used via ReactDOM.render method
==================================================================
getInitialState
componentWillMount
render
componentDidMount
componentWillReceiveProps
shouldComponentUpdate
componentWillUpdate
render
componentDidUpdate
componentWillUnmount
 
Lifecycle of a react component used via ReactTestUtils.renderIntoDocument method
================================================================================
getInitialState
componentWillMount
render
componentDidMount
getInitialState
componentWillMount
render
componentDidMount
componentWillUnmount
componentWillUnmount
```

On s'appercoit que le cycle de vie du composant n'est pas le meme suivant la methode render appelée.


Le programme: [https://jsfiddle.net/JacquesBonet/k0tyztzc/](https://jsfiddle.net/JacquesBonet/k0tyztzc/)
