package com.open.hotel.work;

import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.text.PDFTextStripper;

import java.io.File;
import java.io.IOException;
import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;
import java.util.regex.Pattern;

public class PDF3 {

    public static void main(String[] args) throws IOException {
        String pdf1Path = System.getProperty("user.dir") + "/src/test/resources/pdf/Exception Interview Questions Actual.pdf";
        String pdf2Path = System.getProperty("user.dir") + "/src/test/resources/pdf/Exception Interview Questions Expected.pdf";
        String outputDir = System.getProperty("user.dir") + "/target/";

        comparePDFs(pdf1Path, pdf2Path, outputDir);
    }

    public static void comparePDFs(String pdf1Path, String pdf2Path, String outputDir) throws IOException {
        PDDocument doc1 = PDDocument.load(new File(pdf1Path));
        PDDocument doc2 = PDDocument.load(new File(pdf2Path));

        int numPages = Math.min(doc1.getNumberOfPages(), doc2.getNumberOfPages());

        StringBuilder differences = new StringBuilder();

        for (int i = 0; i < numPages; i++) {
            differences.append(highlightDifferences(doc1, doc2, i));
        }

        doc1.close();
        doc2.close();

        generateHTMLReport(outputDir, numPages, differences.toString());
    }

    private static String highlightDifferences(PDDocument doc1, PDDocument doc2, int pageIndex) throws IOException {
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

        StringBuilder differences = new StringBuilder();
        differences.append("<h3>Page ").append(pageIndex + 1).append(" Differences:</h3>");
        differences.append("<div class='container'>");
        differences.append("<div class='text'><h3>Actual PDF</h3><p>").append(highlightText(text1, missingInDoc2)).append("</p></div>");
        differences.append("<div class='text'><h3>Expected PDF</h3><p>").append(highlightText(text2, missingInDoc1)).append("</p></div>");
        differences.append("<div class='text'><h3>Differences</h3><p>").append(missingInDoc1).append("</p><p>").append(missingInDoc2).append("</p></div>");
        differences.append("</div>");

        return differences.toString();
    }

    private static String highlightText(String text, Set<String> wordsToHighlight) {
        for (String word : wordsToHighlight) {
            String escapedWord = Pattern.quote(word);
            text = text.replaceAll("\\b" + escapedWord + "\\b", "<span style='color:red;'>" + word + "</span>");
        }
        return text;
    }

    private static void generateHTMLReport(String outputDir, int numPages, String differences) throws IOException {
        StringBuilder htmlContent = new StringBuilder();
        htmlContent.append("<html><head><style>")
                .append(".container { display: flex; }")
                .append(".text { width: 30%; margin: 1.5%; }")
                .append("</style></head><body>")
                .append(differences)
                .append("</body></html>");

        File htmlFile = new File(outputDir + "report.html");
        org.apache.commons.io.FileUtils.writeStringToFile(htmlFile, htmlContent.toString(), "UTF-8");
    }
}
