```java/**
class FxRateXmlSupportTest {

    private final FxRateXmlSupport support = new FxRateXmlSupport();

    // ---------------- stripBom ----------------

    @Test
    void stripBom_whenNull_thenReturnEmptyString() {
        assertThat(support.stripBom(null)).isEqualTo("");
    }

    @Test
    void stripBom_whenOnlySpaces_thenReturnEmptyString() {
        assertThat(support.stripBom("   \n\t  ")).isEqualTo("");
    }

    @Test
    void stripBom_whenBomAtBeginning_thenRemoveBomAndTrim() {
        String xml = "\uFEFF   <root/>   ";
        assertThat(support.stripBom(xml)).isEqualTo("<root/>");
    }

    @Test
    void stripBom_whenNoBom_thenJustTrim() {
        String xml = "   <root/>   ";
        assertThat(support.stripBom(xml)).isEqualTo("<root/>");
    }

    @Test
    void stripBom_whenBomNotAtBeginning_thenDoNotRemoveIt() {
        // BOM внутри строки не должен удаляться (replaceFirst только в начале)
        String xml = " <r>\uFEFF</r> ";
        assertThat(support.stripBom(xml)).isEqualTo("<r>\uFEFF</r>");
    }

    // ---------------- extractRoot ----------------

    @Test
    void extractRoot_whenBlankAfterStrip_thenThrowIllegalArgumentException() {
        assertThatThrownBy(() -> support.extractRoot("   \n\t  "))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("XML is blank");
    }

    @Test
    void extractRoot_whenValidXml_thenReturnRootLocalName() {
        String xml = "<PutEODPriceNF><a/></PutEODPriceNF>";
        assertThat(support.extractRoot(xml)).isEqualTo("PutEODPriceNF");
    }

    @Test
    void extractRoot_whenValidXmlWithPrologAndWhitespace_thenReturnRootLocalName() {
        String xml = "   <?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                     "   <root attr=\"1\"><child/></root>";
        assertThat(support.extractRoot(xml)).isEqualTo("root");
    }

    @Test
    void extractRoot_whenValidXmlWithBom_thenReturnRootLocalName() {
        String xml = "\uFEFF  <root><child/></root> ";
        assertThat(support.extractRoot(xml)).isEqualTo("root");
    }

    @Test
    void extractRoot_whenNoRootElement_thenThrowIllegalArgumentExceptionNoRoot() {
        // Пустой документ не blank (есть символы), но START_ELEMENT не встретится
        String xml = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>";
        assertThatThrownBy(() -> support.extractRoot(xml))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("No root element in XML");
    }

    @Test
    void extractRoot_whenInvalidXml_thenThrowIllegalArgumentExceptionWrapped() {
        String xml = "<root>"; // не закрыт

        assertThatThrownBy(() -> support.extractRoot(xml))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessage("Cannot read XML root tag")
            .hasCauseInstanceOf(Exception.class);
    }
}

```
