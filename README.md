## Catalysis

[Catalysis](https://github.com/metasoarous/catalysis) is a full stack micro framework which leverages datalog DBs on the client and server.

    "The acceleration of a chemical reaction by a catalyst."

Currently it uses [datsync](https://github.com/metasoarous/datsync) to sync the Tx reports directly.

We plan to extend catalysis further and make it more flexible, by allowing you to sync affected entities such that validation and security filters can be applied. This gives the app full control over which users get streamed which entities from the same underlying DB, ie. materialized views.

Furthermore, a [datviews]() library is also in the works to help generate views based on the entity schemas. With [Web components](https://www.w3.org/standards/techs/components) and [CSS Grid layout](https://drafts.csswg.org/css-grid/) coming soon, we will finally have full control of the web layout.