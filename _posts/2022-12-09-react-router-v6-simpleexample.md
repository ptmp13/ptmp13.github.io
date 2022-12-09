---
layout:     post
title:      "Example react router simple page (v6)..."
subtitle:   " \"...\""
date:       2022-12-09 11:44:00
author:     "ptmp13"
catalog: true
tags:
    - ReactJS
    - javascript 
---

Ok very simple example with react router.
```bash
npx create-react-app testrq-router && cd $_ && yarn add react-router-dom
```

Modify index.js to:

```js
import React from 'react';
import ReactDOM from 'react-dom/client';
import './index.css';
import { BrowserRouter as Router, Route, Routes } from "react-router-dom";
import { Header, Homepage, Aboutpage } from './App';

const root = ReactDOM.createRoot(document.getElementById("root"));
root.render(
  <Router>
    <Header/>
    <Routes>
      <Route path="/" element={<Homepage />} />
      <Route path="/about" element={<Aboutpage />} />
    <\/Routes>
  <\/Router>
);
```

Modify App.js to:

```js
import logo from './logo.svg';
import './App.css';

import { Link } from 'react-router-dom';

const Header = () => {
    return (
        <div>
                <p>Header<\/p>
        <\/div>
    )
};

const Homepage = () => {
    return (
        <div>
                <h1>Homepage <\/h1>
                <Link to='/about'>Go to Aboutpage<\/Link>
        <\/div>
    )
};

const Aboutpage = () => {
    return (
        <div>
                <h1>Aboutpage<\/h1>
                <Link to='\/'>Go to Homepage<\/Link>
        <\/div>
    )
};
export {Header, Homepage, Aboutpage } ;
```


base on 
https://www.pluralsight.com/guides/understanding-links-in-reactjs?__cf_chl_tk=kGurwoUEUYMaJKzf.DRnlw2Vo5qdPzwqn4FslqtFs2I-1670573139-0-gaNycGzNCKU
mod to react-route v6

Header Component already placed as not rerender component.