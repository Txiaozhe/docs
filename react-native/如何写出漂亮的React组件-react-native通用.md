团队开发时，代码质量尤为重要，优秀的代码通常包含如下特性：

* 即使没有任何注释的情况下也易于理解
* 比乱麻般的代码有更好的性能表现
* 更易于进行bug追溯
* 简洁明了，一句顶一万句

> "Less is more."  —Mies van der Rohe

在react组件开发中常用的一个组件优化模式是Stateless Functional Component（即SFC模式），运用这种模式能有效提高组件的性能表现，且能减少大量的模板代码。

直观来看，SFC就是指仅有一个渲染函数的组件，不过这简单的改变就可以避免很多无意义的检测与内存分配。

若用正统的React组件的写法，可得出以下代码：

```javascript
export default class RelatedSearch extends React.Component {
  constructor(props) {
    super(props);
    this._handleClick = this._handleClick.bind(this);
  }
  _handleClick(suggestedUrl, event) {
    event.preventDefault();
    this.props.onClick(suggestedUrl);
  }
  render() {
    return (
      <section className="related-search-container">
        <h1 className="related-search-title">Related Searches:</h1>
        <Layout x-small={2} small={3} medium={4} padded={true}>
          {this.props.relatedQueries.map((query, index) =>
            <Link
              className="related-search-link"
              onClick={(event) =>
                this._handleClick(query.searchQuery, event)}
              key={index}>
              {query.searchText}
            </Link>
          )}
        </Layout>
      </section>
    );
  }
}
```

而使用SFC模式后，可以节省29%的代码

```javascript
const _handleClick(suggestedUrl, onClick, event) => {
  event.preventDefault();
  onClick(suggestedUrl);
};
const RelatedSearch = ({ relatedQueries, onClick }) =>
  <section className="related-search-container">
    <h1 className="related-search-title">Related Searches:</h1>
    <Layout x-small={2} small={3} medium={4} padded={true}>
      {relatedQueries.map((query, index) =>
        <Link
          className="related-search-link"
          onClick={(event) =>
            _handleClick(query.searchQuery, onClick, event)}
          key={index}>
          {query.searchText}
        </Link>
      )}
    </Layout>
  </section>
export default RelatedSearch;
```

代码的减少主要来源于两个方面：

* 没有构造函数
* 以Arrow Function的方式代替Render语句

事实上，SFC最迷人的地方不仅仅是其代码量的减少，还有就是对于可读性的提高。SFC本身就是所谓纯组件的一种最佳实践范式，而移除了构造函数并且将_handleClick() 这个点击事件回调函数提取出组件外，可以使JSX代码变得更加纯粹。另一个不错的地方就是SFC以Arrow Function的方式来定义输入的props变量，即以Object Destructing语法来声明组件所依赖的Props：

```javascript
const RelatedSearch = ({ relatedQueries, onClick }) =>
```

这样不仅能够使组件的Props更加清晰明确，还能避免冗余的this.props表达式，从而使代码的可读性更好。

然而有以下特征的组件不适合使用SFC：

* 需要自定义整个组件的生命周期管理
* 需要使用到refs

JSX本身不支持if表达式，不过可以使用逻辑表达式的方式来避免将代码切分到不同的子模块中，如下所示：

```javascript
render() {
  <div class="search-results-container">
    {this.props.isGrid
      ? <SearchResultsGrid />
      : <SearchResultsList />}
  </div>
}
```

这种表达式在二选一渲染的时候很有效果，不过对于选择性渲染一个的情况很不友好：

```javascript
render() {
  <div class="search-results-list">
    {this.props.isSoftSort
      ? <SoftSortBanner />
      : null
    }
  </div>
}
```

但也可以选用另一种更加语义化与友好的方式来实现这个功能，即使用逻辑与表达式然后返回组件：

```javascript
render() {
  <div class="search-results-list">
    {!!this.props.isSoftSort && <SoftSortBanner />}
  </div>
}
```

Arrow Function

```javascript
const SoftSort = ({ hardSortUrl, sortByName, onClick }) => {
  return (
    <div className="SearchInfoMessage">
      Showing results sorted by both Relevance and {sortByName}.
      <Link
        href={`?${hardSortUrl}`}
        onClick={(ev) => onClick(ev, hardSortUrl)}>
        Sort results by {sortByName} only
      </Link>
    </div>
  );
};
```

该函数的功能就是返回JSX对象，也可以忽略return语句

```javascript
const SoftSort = ({ hardSortUrl, sortByName, onClick }) =>
  <div className="SearchInfoMessage">
    Showing results sorted by both Relevance and {sortByName}.
    <Link
      href={`?${hardSortUrl}`}
      onClick={(ev) => onClick(ev, hardSortUrl)}>
      Sort results by {sortByName} only
    </Link>
  </div>
```

需要注意的是，如果返回的是Object，需要包裹在大括号内：

```javascript
const mapStateToProps = ({isLoading}) => ({
  loading: isLoading
});
```

使用Arrow Function优化的核心点在于其能够通过专注于函数的重要部分而提升代码的整体可读性，并且避免过多的模板代码带来的噪音

大的组件往往受困于this.props过长的窘境，典型的如下所示:

```javascript
render() {
  return (
    <ProductPrice
      hidePriceFulfillmentDisplay=
       {this.props.hidePriceFulfillmentDisplay}
      primaryOffer={this.props.primaryOffer}
      productType={this.props.productType}
      productPageUrl={this.props.productPageUrl}
      inventory={this.props.inventory}
      submapType={this.props.submapType}
      ppu={this.props.ppu}
      isLoggedIn={this.props.isLoggedIn}
      gridView={this.props.isGridView}
    />
  );
}
```

如果要将这些Props继续传入下一层，就要变成下面这个样子:

```javascript
render() {
  const {
    hidePriceFulfillmentDisplay,
    primaryOffer,
    productType,
    productPageUrl,
    inventory,
    submapType,
    ppu,
    isLoggedIn,
    gridView
  } = this.props;
  return (
    <ProductPrice
      hidePriceFulfillmentDisplay={hidePriceFulfillmentDisplay}
      primaryOffer={primaryOffer}
      productType={productType}
      productPageUrl={productPageUrl}
      inventory={inventory}
      submapType={submapType}
      ppu={ppu}
      isLoggedIn={isLoggedIn}
      gridView={isGridView}
    />
  );
}
```

可以使用解构赋值来实现这个功能：

```javascript
render() {
  const props = this.props;
  return <ProductPrice {...props} />
}
```

如果需要在Object中添加函数，你可以使用ES2015 Method Definition Shorthand来代替传统的ES5的表达式，如:

```javascript
ProductRating.defaultProps = {
  onStarsClick() {}
};
```

如果需要设置一个默认的空方法，也可以利用这种方式。











