package com.open.hotel.work;

import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.PDPage;
import org.apache.pdfbox.pdmodel.PDPageContentStream;
import org.apache.pdfbox.pdmodel.common.PDRectangle;
import org.apache.pdfbox.text.PDFTextStripper;
import org.apache.pdfbox.rendering.PDFRenderer;
import org.apache.pdfbox.tools.imageio.ImageIOUtil;

import java.awt.*;
import java.awt.image.BufferedImage;
import java.io.File;
import java.io.IOException;
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

public class PDF2 {

    public static void main(String[] args) throws IOException {
        String pdf1Path = System.getProperty("user.dir") + "/src/test/resources/pdf/Exception Interview Questions Actual.pdf";
        String pdf2Path = System.getProperty("user.dir") + "/src/test/resources/pdf/Exception Interview Questions Expected.pdf";
        String outputDir = System.getProperty("user.dir") + "/target/";

        comparePDFs(pdf1Path, pdf2Path, outputDir);
    }

    public static void comparePDFs(String pdf1Path, String pdf2Path, String outputDir) throws IOException {
        PDDocument doc1 = PDDocument.load(new File(pdf1Path));
        PDDocument doc2 = PDDocument.load(new File(pdf2Path));

        PDFRenderer pdfRenderer1 = new PDFRenderer(doc1);
        PDFRenderer pdfRenderer2 = new PDFRenderer(doc2);

        int numPages = Math.min(doc1.getNumberOfPages(), doc2.getNumberOfPages());

        for (int i = 0; i < numPages; i++) {
            highlightDifferences(doc1, doc2, i);

            BufferedImage image1 = pdfRenderer1.renderImageWithDPI(i, 300);
            BufferedImage image2 = pdfRenderer2.renderImageWithDPI(i, 300);
            BufferedImage diffImage = getDifferenceImage(image1, image2);

            ImageIOUtil.writeImage(image1, outputDir + "page" + (i + 1) + "_original1.png", 300);
            ImageIOUtil.writeImage(image2, outputDir + "page" + (i + 1) + "_original2.png", 300);
            ImageIOUtil.writeImage(diffImage, outputDir + "page" + (i + 1) + "_diff.png", 300);
        }

        doc1.close();
        doc2.close();

        generateHTMLReport(outputDir, numPages);
    }

    private static void highlightDifferences(PDDocument doc1, PDDocument doc2, int pageIndex) throws IOException {
        PDFTextStripper textStripper = new PDFTextStripper();
        textStripper.setStartPage(pageIndex + 1);
        textStripper.setEndPage(pageIndex + 1);

        String text1 = textStripper.getText(doc1);
        String text2 = textStripper.getText(doc2);

        Set<String> words1 = new HashSet<>(Arrays.asList(text1.split("\\s+")));
        Set<String> words2 = new HashSet<>(Arrays.asList(text2.split("\\s+")));
        Set<String> missingInDoc1 = new HashSet<>(words2);
        missingInDoc1.removeAll(words1);

        Set<String> missingInDoc2 = new HashSet<>(words1);
        missingInDoc2.removeAll(words2);

        PDPage page1 = doc1.getPage(pageIndex);
        PDPage page2 = doc2.getPage(pageIndex);

        highlightWords(doc1, page1, missingInDoc1);
        highlightWords(doc2, page2, missingInDoc2);
    }

    private static void highlightWords(PDDocument doc, PDPage page, Set<String> words) throws IOException {
        PDRectangle mediaBox = page.getMediaBox();
        try (PDPageContentStream contentStream = new PDPageContentStream(doc, page, PDPageContentStream.AppendMode.APPEND, true, true)) {
            contentStream.setStrokingColor(Color.RED);
            contentStream.setLineWidth(1.0f);

            for (String word : words) {
                // This is a simplified example. In a real scenario, you would need to find the exact position of the word on the page.
                float x = mediaBox.getLowerLeftX() + 50;
                float y = mediaBox.getUpperRightY() - 50;
                float width = 100;
                float height = 20;

                contentStream.addRect(x, y, width, height);
                contentStream.stroke();
            }
        }
    }

    private static BufferedImage getDifferenceImage(BufferedImage img1, BufferedImage img2) {
        int width = img1.getWidth();
        int height = img1.getHeight();
        BufferedImage diffImage = new BufferedImage(width, height, BufferedImage.TYPE_INT_RGB);
        Graphics2D g = diffImage.createGraphics();
        g.drawImage(img1, 0, 0, null);

        g.setColor(Color.RED);
        for (int y = 0; y < height; y++) {
            for (int x = 0; x < width; x++) {
                int rgb1 = img1.getRGB(x, y);
                int rgb2 = img2.getRGB(x, y);
                if (rgb1 != rgb2) {
                    g.drawLine(x, y, x, y);
                }
            }
        }
        g.dispose();
        return diffImage;
    }

    private static void generateHTMLReport(String outputDir, int numPages) throws IOException {
        StringBuilder htmlContent = new StringBuilder();
        htmlContent.append("<html><head><style>")
                .append(".container { display: flex; }")
                .append(".image { width: 30%; margin: 1.5%; cursor: pointer; }")
                .append(".image img { width: 100%; }")
                .append(".modal { display: none; position: fixed; z-index: 1; padding-top: 60px; left: 0; top: 0; width: 100%; height: 100%; overflow: auto; background-color: rgb(0,0,0); background-color: rgba(0,0,0,0.9); }")
                .append(".modal-content { margin: auto; display: block; width: 80%; max-width: 700px; }")
                .append(".modal-content, .caption { animation-name: zoom; animation-duration: 0.6s; }")
                .append("@keyframes zoom { from {transform:scale(0)} to {transform:scale(1)} }")
                .append(".close { position: absolute; top: 15px; right: 35px; color: #f1f1f1; font-size: 40px; font-weight: bold; transition: 0.3s; }")
                .append(".close:hover, .close:focus { color: #bbb; text-decoration: none; cursor: pointer; }")
                .append("</style></head><body>");

        for (int i = 0; i < numPages; i++) {
            htmlContent.append("<h2>Page ").append(i + 1).append("</h2>")
                    .append("<div class='container'>")
                    .append("<div class='image'><h3>Actual PDF</h3><img src='page").append(i + 1).append("_original1.png' onclick='openModal(this)'/></div>")
                    .append("<div class='image'><h3>Expected PDF</h3><img src='page").append(i + 1).append("_original2.png' onclick='openModal(this)'/></div>")
                    .append("<div class='image'><h3>Difference PDF</h3><img src='page").append(i + 1).append("_diff.png' onclick='openModal(this)'/></div>")
                    .append("</div>");
        }

        htmlContent.append("<div id='myModal' class='modal'>")
                .append("<span class='close' onclick='closeModal()'>&times;</span>")
                .append("<img class='modal-content' id='img01'>")
                .append("</div>")
                .append("<script>")
                .append("function openModal(img) {")
                .append("var modal = document.getElementById('myModal');")
                .append("var modalImg = document.getElementById('img01');")
                .append("modal.style.display = 'block';")
                .append("modalImg.src = img.src;")
                .append("}")
                .append("function closeModal() {")
                .append("var modal = document.getElementById('myModal');")
                .append("modal.style.display = 'none';")
                .append("}")
                .append("</script>")
                .append("</body></html>");

        File htmlFile = new File(outputDir + "report.html");
        org.apache.commons.io.FileUtils.writeStringToFile(htmlFile, htmlContent.toString(), "UTF-8");
    }
}
